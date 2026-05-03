Segment WooCommerce Customers for Targeted Marketing with RFM Analysis

https://n8nworkflows.xyz/workflows/segment-woocommerce-customers-for-targeted-marketing-with-rfm-analysis-4880


# Segment WooCommerce Customers for Targeted Marketing with RFM Analysis

### 1. Workflow Overview

This workflow automates the process of segmenting WooCommerce customers based on their purchasing behavior using RFM (Recency, Frequency, Monetary) analysis. It targets e-commerce marketers and business analysts who want to identify customer groups for targeted marketing campaigns, improving customer engagement and sales efficiency.

The workflow is organized into the following logical blocks:

- **1.1 Input Reception and Scheduling:** Receives manual or scheduled triggers to start the workflow.
- **1.2 Data Acquisition:** Fetches all completed WooCommerce orders from the past year.
- **1.3 RFM Computation:** Processes order data to compute RFM scores and assigns customers to defined segments.
- **1.4 Summary Generation:** Creates an HTML summary report of customer segments with marketing suggestions.
- **1.5 Report Formatting:** Builds a styled HTML page for visualization or distribution.
- **1.6 Setup Guidance:** Provides instructions for configuring the workflow before use.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Scheduling

**Overview:**  
This block triggers the workflow either manually for testing purposes or on a recurring weekly schedule.

**Nodes Involved:**  
- Manual Start  
- Weekly Trigger

**Node Details:**

- **Manual Start**  
  - Type: Start node (manual trigger)  
  - Role: Allows manual initiation of the workflow for testing or ad-hoc runs.  
  - Configuration: Default, no parameters.  
  - Inputs: None  
  - Outputs: Connected to "Fetch Orders From WooCommerce"  
  - Edge Cases: None; manual trigger is user-initiated.

- **Weekly Trigger**  
  - Type: Schedule Trigger  
  - Role: Automatically triggers the workflow weekly to keep segmentation updated.  
  - Configuration: Interval set to weekly (every 1 week).  
  - Inputs: None  
  - Outputs: Connected to "Fetch Orders From WooCommerce"  
  - Edge Cases: Potential issues if the workflow runtime exceeds schedule interval; no authentication needed.

#### 2.2 Data Acquisition

**Overview:**  
Fetches all completed orders from WooCommerce created within the last year, ensuring that data for RFM analysis is current and comprehensive.

**Nodes Involved:**  
- Fetch Orders From WooCommerce

**Node Details:**

- **Fetch Orders From WooCommerce**  
  - Type: WooCommerce node (API integration)  
  - Role: Retrieves all completed orders from WooCommerce created after a dynamic date (one year ago from current date).  
  - Configuration:  
    - Resource: Order  
    - Operation: Get All  
    - Status filter: completed orders only  
    - After date: dynamic expression `{{$now.minus(1, 'year').toISO()}}`  
    - Return all results: true  
  - Credentials: WooCommerce API credentials (must be configured with valid keys)  
  - Inputs: From Manual Start or Weekly Trigger  
  - Outputs: To "Compute RFM Segmentation"  
  - Edge Cases:  
    - API authentication errors if credentials invalid  
    - Large data sets may cause timeouts or rate limits  
    - Network failures  

#### 2.3 RFM Computation

**Overview:**  
Processes fetched orders to compute Recency (days since last purchase), Frequency (number of orders), Monetary (total spent), calculates quartile-based RFM scores, assigns customer segments, and calculates average order value.

**Nodes Involved:**  
- Compute RFM Segmentation

**Node Details:**

- **Compute RFM Segmentation**  
  - Type: Code node (JavaScript)  
  - Role: Implements RFM scoring logic and segmentation classification.  
  - Configuration Highlights:  
    - Groups orders by customer ID or billing email  
    - Calculates recency as days since last order  
    - Frequency as count of orders  
    - Monetary as total spent  
    - Sorts customers by R, F, M values to determine quartiles  
    - Assigns R, F, M scores (1-4) with inverted scales where appropriate  
    - Defines segments such as Champions, Loyal Customers, Potential Loyalists, New Customers, etc.  
    - Calculates average order value per customer  
  - Key Expressions: Uses JavaScript Date and array methods, custom quartile calculation function  
  - Inputs: Orders from WooCommerce node  
  - Outputs: Array of customer objects with RFM scores and segments to "Generate Segment Summary"  
  - Edge Cases:  
    - Missing customer ID or email in orders (skipped)  
    - Orders with invalid or missing dates or totals may cause calculation errors  
    - Division by zero if frequency is zero (should not occur as frequency counts orders)  
    - Large data sets may impact performance  
  - Version Specifics: Uses typeVersion 2 for code node (supporting ES6+)

#### 2.4 Summary Generation

**Overview:**  
Aggregates customers by segment, counts customers per segment, and produces an HTML table with segments, counts, and recommended marketing actions.

**Nodes Involved:**  
- Generate Segment Summary

**Node Details:**

- **Generate Segment Summary**  
  - Type: Code node (JavaScript)  
  - Role: Builds an HTML summary report with segment counts and marketing suggestions.  
  - Configuration Highlights:  
    - Defines segment labels, priorities, and marketing actions  
    - Counts customers per segment from input data  
    - Sorts segments by priority for presentation  
    - Produces HTML code with a styled table  
  - Inputs: Customer RFM data from previous node  
  - Outputs: JSON object containing `html` string to "Build Report Page"  
  - Edge Cases:  
    - Segment values missing or unrecognized default to 'Others'  
    - Empty input results in empty table body  
  - Version Specifics: Uses typeVersion 2 code node

#### 2.5 Report Formatting

**Overview:**  
Formats the generated HTML into a styled webpage for easy visualization or email delivery.

**Nodes Involved:**  
- Build Report Page

**Node Details:**

- **Build Report Page**  
  - Type: HTML node  
  - Role: Wraps the generated HTML summary with CSS styling for professional presentation.  
  - Configuration:  
    - Static HTML head with styles for table and text formatting  
    - Inserts dynamic content `{{ $json.html }}` from previous node  
  - Inputs: HTML string from "Generate Segment Summary"  
  - Outputs: Formatted HTML page (can be sent via email or viewed)  
  - Edge Cases: None significant, but malformed HTML input could break display  
  - Version Specifics: Uses typeVersion 1.2

#### 2.6 Setup Guidance

**Overview:**  
Provides users with setup instructions and recommendations for configuring the workflow before running.

**Nodes Involved:**  
- Sticky Note

**Node Details:**

- **Sticky Note**  
  - Type: Sticky Note (documentation)  
  - Role: Displays setup and usage instructions within the n8n editor.  
  - Content Summary:  
    - Reminder to configure WooCommerce credentials  
    - Optional customization points for RFM logic and marketing suggestions  
    - Instructions on manual and scheduled triggering  
  - Inputs/Outputs: None (informational only)  
  - Edge Cases: None

---

### 3. Summary Table

| Node Name                   | Node Type            | Functional Role                          | Input Node(s)          | Output Node(s)            | Sticky Note                                                                                      |
|-----------------------------|----------------------|----------------------------------------|------------------------|---------------------------|-------------------------------------------------------------------------------------------------|
| Manual Start                | Start                | Manual workflow trigger                 | None                   | Fetch Orders From WooCommerce | See setup instructions in Sticky Note for usage details                                        |
| Weekly Trigger             | Schedule Trigger      | Scheduled weekly trigger                | None                   | Fetch Orders From WooCommerce | See setup instructions in Sticky Note for usage details                                        |
| Fetch Orders From WooCommerce | WooCommerce          | Retrieve completed orders from WooCommerce within last year | Manual Start, Weekly Trigger | Compute RFM Segmentation     | Must configure WooCommerce credentials as per Sticky Note                                       |
| Compute RFM Segmentation    | Code                 | Calculate RFM scores and assign segments | Fetch Orders From WooCommerce | Generate Segment Summary    | Customizable segmentation logic; see Sticky Note                                               |
| Generate Segment Summary    | Code                 | Create HTML summary report of segments | Compute RFM Segmentation | Build Report Page          | Marketing suggestions customizable; see Sticky Note                                            |
| Build Report Page           | HTML                 | Format HTML summary with CSS styling    | Generate Segment Summary | None                      |                                                                                                 |
| Sticky Note                | Sticky Note           | Setup instructions and configuration tips | None                   | None                      | Contains setup and configuration instructions for the entire workflow                          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Nodes**  
   - Add a **Start** node named "Manual Start" with default settings.  
   - Add a **Schedule Trigger** node named "Weekly Trigger", set interval to weekly (every 1 week).

2. **Add WooCommerce Data Fetch Node**  
   - Add a **WooCommerce** node named "Fetch Orders From WooCommerce".  
   - Configure with:  
     - Resource: Order  
     - Operation: Get All  
     - Status: completed  
     - Options → After: Set to expression `{{$now.minus(1, 'year').toISO()}}` (fetch orders from last year).  
   - Attach WooCommerce API credentials (set up a credential with your WooCommerce API keys).  
   - Connect outputs of "Manual Start" and "Weekly Trigger" to this node.

3. **Add RFM Computation Node**  
   - Add a **Code** node named "Compute RFM Segmentation".  
   - Set code language to JavaScript and paste the provided RFM calculation script.  
   - Connect output of "Fetch Orders From WooCommerce" to this node.

4. **Add Segment Summary Generation Node**  
   - Add a **Code** node named "Generate Segment Summary".  
   - Paste the JavaScript code that builds an HTML table summarizing segment counts and marketing suggestions.  
   - Connect output of "Compute RFM Segmentation" to this node.

5. **Add HTML Formatting Node**  
   - Add an **HTML** node named "Build Report Page".  
   - Paste the provided HTML template with CSS styles that wraps the summary table.  
   - Use expression `{{ $json.html }}` to inject the HTML from the previous node.  
   - Connect output of "Generate Segment Summary" to this node.

6. **Add Setup Instructions Node (Optional)**  
   - Add a **Sticky Note** node named "Sticky Note".  
   - Enter the setup instructions content describing credential setup, customization options, and usage tips.

7. **Verify Connections:**  
   - Connect "Manual Start" → "Fetch Orders From WooCommerce"  
   - Connect "Weekly Trigger" → "Fetch Orders From WooCommerce"  
   - Connect "Fetch Orders From WooCommerce" → "Compute RFM Segmentation"  
   - Connect "Compute RFM Segmentation" → "Generate Segment Summary"  
   - Connect "Generate Segment Summary" → "Build Report Page"

8. **Configure Credentials:**  
   - Create and select valid WooCommerce API credentials with read access to orders.

9. **Test Workflow:**  
   - Run manually via "Manual Start" to verify data retrieval and report generation.  
   - Review HTML output in "Build Report Page" node.

10. **Enable Weekly Automation:**  
    - Activate the workflow with the schedule trigger enabled for weekly execution.

---

### 5. General Notes & Resources

| Note Content                                                                                                            | Context or Link                         |
|-------------------------------------------------------------------------------------------------------------------------|---------------------------------------|
| The RFM segmentation logic can be customized inside the "Compute RFM Segmentation" node by editing the JavaScript code. | Customization point for segmentation logic |
| Marketing suggestions for each segment are editable in the "Generate Segment Summary" node's JavaScript.                 | Customization point for marketing actions |
| Ensure WooCommerce API credentials have appropriate permissions for order read access.                                   | WooCommerce API documentation         |
| Setup instructions are provided inside the Sticky Note node for user convenience.                                        | Workflow internal documentation       |
| For more on RFM analysis and segmentation strategies, consult external marketing analytics resources.                   | External marketing analytics blogs or sites |

---

**Disclaimer:**  
The provided text is exclusively derived from an automated n8n workflow. All processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data handled is lawful and publicly accessible.