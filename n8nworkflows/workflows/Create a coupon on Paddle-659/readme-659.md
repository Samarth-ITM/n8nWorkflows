Create a coupon on Paddle

https://n8nworkflows.xyz/workflows/create-a-coupon-on-paddle-659


# Create a coupon on Paddle

### 1. Workflow Overview

This workflow automates the creation of a discount coupon on the Paddle platform. It is designed for users who want to programmatically generate a coupon with predefined parameters such as discount amount and coupon code. The workflow consists of two main logical blocks:

- **1.1 Manual Trigger:** Waits for user initiation to start the coupon creation process.
- **1.2 Paddle API Coupon Creation:** Connects to the Paddle API to create a coupon with specified discount details.

---

### 2. Block-by-Block Analysis

#### 1.1 Manual Trigger

- **Overview:**  
  This block serves as the entry point for the workflow. It waits for manual execution by the user, allowing controlled initiation of the coupon creation process.

- **Nodes Involved:**  
  - On clicking 'execute'

- **Node Details:**  
  - **Node Name:** On clicking 'execute'  
  - **Type and Technical Role:** Manual Trigger node; triggers the workflow manually on user command.  
  - **Configuration Choices:** Default configuration with no parameters required.  
  - **Key Expressions or Variables:** None, as this is a trigger node.  
  - **Input and Output Connections:**  
    - Input: None (trigger node).  
    - Output: Connected to the "Paddle" node.  
  - **Version-Specific Requirements:** Compatible with n8n version 1.x and above.  
  - **Edge Cases or Potential Failures:**  
    - If the workflow is inactive, the trigger will not respond.  
    - No authentication needed here, so limited failure modes.  
  - **Sub-Workflow Reference:** None.

#### 1.2 Paddle API Coupon Creation

- **Overview:**  
  This block handles the creation of a coupon on Paddle using the Paddle API node. The coupon details such as discount amount and coupon code are statically configured in the node.

- **Nodes Involved:**  
  - Paddle

- **Node Details:**  
  - **Node Name:** Paddle  
  - **Type and Technical Role:** Paddle node; interfaces with Paddle's API to create or manage coupons.  
  - **Configuration Choices:**  
    - Discount Amount: 2 (currency units, e.g., dollars).  
    - Additional Fields: Coupon Code set to "n8n-docs".  
    - Credentials: Uses stored Paddle API credentials under the name "paddleApi".  
  - **Key Expressions or Variables:** Static values for discountAmount and couponCode; no dynamic expressions used.  
  - **Input and Output Connections:**  
    - Input: Receives trigger from "On clicking 'execute'".  
    - Output: No further nodes connected, ends workflow.  
  - **Version-Specific Requirements:** Requires Paddle API credentials configured in n8n prior to execution. Compatible with n8n version 1.x and above.  
  - **Edge Cases or Potential Failures:**  
    - Authentication errors if Paddle API credentials are invalid or expired.  
    - API rate limits or service downtime causing request failures.  
    - Invalid coupon parameters (e.g., coupon code conflicts) resulting in API errors.  
    - Network connectivity issues.  
  - **Sub-Workflow Reference:** None.

---

### 3. Summary Table

| Node Name           | Node Type             | Functional Role          | Input Node(s)       | Output Node(s) | Sticky Note                                |
|---------------------|-----------------------|-------------------------|---------------------|----------------|--------------------------------------------|
| On clicking 'execute'| Manual Trigger        | Start workflow manually | None                | Paddle         |                                            |
| Paddle              | Paddle API            | Create coupon on Paddle | On clicking 'execute'| None           | Use credentials named "paddleApi" for auth.|

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node:**  
   - Add a "Manual Trigger" node.  
   - Leave default settings (no parameters needed).  
   - This node will start the workflow manually when clicked.

2. **Create Paddle Node:**  
   - Add a "Paddle" node.  
   - Set **Discount Amount** to `2`. This represents the coupon discount value.  
   - Open **Additional Fields** section and set **Coupon Code** to `"n8n-docs"`.  
   - Assign Paddle API credentials:  
     - Create or select existing Paddle API credentials in n8n.  
     - Ensure credentials have valid API key and permissions to create coupons.  
   - This node will call Paddle's API to create the coupon.

3. **Connect Nodes:**  
   - Link the output of "Manual Trigger" to the input of the "Paddle" node.

4. **Activate and Test:**  
   - Save the workflow.  
   - Activate the workflow if desired.  
   - Click the "Execute" button on the manual trigger node to run the workflow and create the coupon on Paddle.

---

### 5. General Notes & Resources

| Note Content                                                                                                   | Context or Link                                   |
|---------------------------------------------------------------------------------------------------------------|--------------------------------------------------|
| Ensure Paddle API credentials are correctly set up and have permissions to create coupons.                     | Paddle API documentation: https://paddle.com/docs/api-reference/ |
| Coupon code "n8n-docs" is statically set; to create dynamic coupons, consider using expressions or variables. | n8n Expressions Docs: https://docs.n8n.io/nodes/expressions/ |
| This workflow is a minimal example and can be extended with error handling or dynamic input parameters.       | n8n Community Forum: https://community.n8n.io/   |

---

This document fully describes the "Create a coupon on Paddle" workflow, enabling reproduction, modification, and troubleshooting.