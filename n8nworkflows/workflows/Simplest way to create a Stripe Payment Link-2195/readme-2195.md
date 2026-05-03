Simplest way to create a Stripe Payment Link

https://n8nworkflows.xyz/workflows/workflow-2195-1747220165388.png


# Simplest way to create a Stripe Payment Link

### 1. Workflow Overview

This workflow provides the simplest and fastest way to create a Stripe Payment Link using n8n. It is designed for businesses or individuals who want to accept online credit card payments through Stripe without the complexity of building a full payment system. The workflow guides users from inputting a product title and price to instantly generating a Stripe payment link that can be shared with customers.

**Logical Blocks:**

- **1.1 Input Reception:** Captures user input (product title and price) via a web form.
- **1.2 Configuration:** Adjusts the input data by applying currency and price formatting.
- **1.3 Stripe Product Creation:** Creates a Stripe product with the given title, price, and currency.
- **1.4 Stripe Payment Link Creation:** Generates a Stripe payment link based on the created product.
- **1.5 Response Handling:** Redirects the user to the created payment link URL.
- **1.6 Setup Notes:** Provides guidance on credential setup and workflow usage.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
This block collects the required input from the user—the product title and price—via a web form triggered by an HTTP request.

- **Nodes Involved:**  
  - Creation Form

- **Node Details:**

  - **Creation Form**  
    - *Type & Role:* Form Trigger node; entry point for user input.  
    - *Configuration:*  
      - Webhook path: `my-form-id`  
      - Form fields:  
        - `title` (text, required)  
        - `price` (number, required)  
      - Response mode: Respond with node (enables downstream control of response).  
    - *Input/Output:* No input; outputs JSON with form data fields `title` and `price`.  
    - *Potential Failures:* Webhook not reachable; invalid or missing form data; network issues.  
    - *Version:* 2

---

#### 1.2 Configuration

- **Overview:**  
Prepares and formats the input data for Stripe API consumption by setting the currency and converting the price to the smallest currency unit (e.g., cents).

- **Nodes Involved:**  
  - Config

- **Node Details:**

  - **Config**  
    - *Type & Role:* Set node; modifies and enriches incoming data.  
    - *Configuration:*  
      - Sets `currency` field statically to `"EUR"` (can be changed as per user needs).  
      - Computes `price` as `price * 100` to convert from a decimal number (e.g., 10.99) to an integer value in cents (e.g., 1099).  
      - Includes original `title` and any other fields unchanged.  
    - *Input/Output:* Input from Creation Form; outputs JSON including `title`, computed `price` (in cents), and `currency`.  
    - *Potential Failures:* Incorrect arithmetic expression; missing input fields.  
    - *Version:* 3.3  
    - *Notes:* User must edit this node to set the desired currency.

---

#### 1.3 Stripe Product Creation

- **Overview:**  
Creates a new product in Stripe with the given title and pricing information, including the default price data.

- **Nodes Involved:**  
  - Create Stripe Product

- **Node Details:**

  - **Create Stripe Product**  
    - *Type & Role:* HTTP Request node; performs POST request to Stripe API to create a product.  
    - *Configuration:*  
      - URL: `https://api.stripe.com/v1/products`  
      - Method: POST  
      - Authentication: Uses stored Stripe API credentials (`stripeApi`) with OAuth or API key.  
      - Body parameters (form-urlencoded):  
        - `name` = product title (`{{$json.title}}`)  
        - `default_price_data[unit_amount]` = product price in smallest currency unit (`{{$json.price}}`)  
        - `default_price_data[currency]` = currency code (`{{$json.currency}}`)  
    - *Input/Output:* Takes JSON from Config node; outputs Stripe product JSON response including `default_price` ID.  
    - *Potential Failures:* Authentication errors, Stripe API rate limits, invalid parameters, network timeouts.  
    - *Version:* 4.1

---

#### 1.4 Stripe Payment Link Creation

- **Overview:**  
Generates a Stripe payment link for the product created in the previous step.

- **Nodes Involved:**  
  - Create payment link

- **Node Details:**

  - **Create payment link**  
    - *Type & Role:* HTTP Request node; POSTs to Stripe payment links API endpoint.  
    - *Configuration:*  
      - URL: `https://api.stripe.com/v1/payment_links`  
      - Method: POST  
      - Authentication: Same Stripe credentials as previous node.  
      - Body parameters (form-urlencoded):  
        - `line_items[0][price]` = price ID from previous node (`{{$json.default_price}}`)  
        - `line_items[0][quantity]` = 1 (fixed quantity)  
    - *Input/Output:* Input is product creation response; outputs payment link JSON including `url`.  
    - *Potential Failures:* Invalid price ID, authentication failure, Stripe API issues.  
    - *Version:* 4.1

---

#### 1.5 Response Handling

- **Overview:**  
Redirects the user to the generated Stripe payment link URL, completing the flow.

- **Nodes Involved:**  
  - Respond to Webhook

- **Node Details:**

  - **Respond to Webhook**  
    - *Type & Role:* Respond to Webhook node; sends HTTP response back to the form submitter.  
    - *Configuration:*  
      - Respond with `redirect` mode.  
      - Redirect URL is dynamically set to the generated Stripe payment link URL (`{{$json.url}}`).  
    - *Input/Output:* Input from Create payment link node; sends HTTP redirect response to the user.  
    - *Potential Failures:* Missing or malformed URL, network errors.  
    - *Version:* 1

---

#### 1.6 Setup Notes

- **Overview:**  
Provides setup instructions and credits for users setting up or modifying the workflow.

- **Nodes Involved:**  
  - Sticky Note1  
  - Sticky Note

- **Node Details:**

  - **Sticky Note1**  
    - *Type & Role:* Sticky Note; contains setup instructions.  
    - *Content:*  
      - Reminds to add Stripe credentials with link to n8n docs.  
      - Instructs to fill in the Config node for currency setup.

  - **Sticky Note**  
    - *Type & Role:* Sticky Note; credits the workflow author.  
    - *Content:* Crafted by [🥷 n8n.ninja](https://n8n.ninja).

---

### 3. Summary Table

| Node Name            | Node Type              | Functional Role                | Input Node(s)        | Output Node(s)           | Sticky Note                                                   |
|----------------------|------------------------|-------------------------------|----------------------|--------------------------|---------------------------------------------------------------|
| Creation Form        | Form Trigger           | Input Reception: Capture user product title and price | None                 | Config                   |                                                               |
| Config               | Set                    | Prepare and format input (currency, price in cents)  | Creation Form        | Create Stripe Product     | Setup your flow; set currency (Sticky Note1)                  |
| Create Stripe Product| HTTP Request           | Create a Stripe product with price and currency       | Config               | Create payment link       |                                                               |
| Create payment link  | HTTP Request           | Create Stripe payment link for the product             | Create Stripe Product| Respond to Webhook        |                                                               |
| Respond to Webhook   | Respond to Webhook     | Redirect user to Stripe payment link URL               | Create payment link  | None                     |                                                               |
| Sticky Note1         | Sticky Note            | Setup instructions                                        | None                 | None                     | # Setup 1/ Add Your credentials [Stripe](https://docs.n8n.io/integrations/builtin/credentials/stripe/) 2/ Fill Config node |
| Sticky Note          | Sticky Note            | Workflow credits                                          | None                 | None                     | Crafted by [🥷 n8n.ninja](https://n8n.ninja)                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the “Creation Form” Node:**  
   - Type: Form Trigger  
   - Configuration:  
     - Path: `my-form-id`  
     - Form Title: "Create a payment link"  
     - Form Fields:  
       - `title` (text, required)  
       - `price` (number, required)  
     - Response Mode: “responseNode”  
   - No credentials required.

2. **Create the “Config” Node:**  
   - Type: Set node  
   - Configuration:  
     - Include field: `title` (from previous node) and any other fields.  
     - Assign static field `currency` with value `"EUR"` (change as needed).  
     - Assign field `price` with expression: `{{$json.price * 100}}` to convert to cents.  
   - Connect the output of “Creation Form” to “Config”.

3. **Set up Stripe credentials in n8n:**  
   - Go to Credentials → New → Stripe API.  
   - Add your Stripe API key or OAuth credentials.  
   - Name the credential “Stripe Dev” or appropriate.

4. **Create the “Create Stripe Product” Node:**  
   - Type: HTTP Request  
   - Configuration:  
     - URL: `https://api.stripe.com/v1/products`  
     - Method: POST  
     - Authentication: Use Stripe API credentials created above.  
     - Content Type: `form-urlencoded`  
     - Body Parameters:  
       - `name` → Expression: `{{$json.title}}`  
       - `default_price_data[unit_amount]` → Expression: `{{$json.price}}`  
       - `default_price_data[currency]` → Expression: `{{$json.currency}}`  
   - Connect “Config” node output to this node.

5. **Create the “Create payment link” Node:**  
   - Type: HTTP Request  
   - Configuration:  
     - URL: `https://api.stripe.com/v1/payment_links`  
     - Method: POST  
     - Authentication: Use same Stripe credentials.  
     - Content Type: `form-urlencoded`  
     - Body Parameters:  
       - `line_items[0][price]` → Expression: `{{$json.default_price}}` (from previous node output)  
       - `line_items[0][quantity]` → Value: `1`  
   - Connect output of “Create Stripe Product” node to this node.

6. **Create the “Respond to Webhook” Node:**  
   - Type: Respond to Webhook  
   - Configuration:  
     - Respond with: Redirect  
     - Redirect URL: Expression: `{{$json.url}}` (payment link URL from previous node)  
   - Connect output of “Create payment link” node to this node.

7. **Add Sticky Notes:** (Optional but recommended)  
   - Create a sticky note with setup instructions including credential setup and config node customization.  
   - Create a sticky note with workflow credits linking to n8n.ninja.

8. **Test the Workflow:**  
   - Activate the workflow.  
   - Access the form via your n8n webhook URL (e.g., `https://your-n8n-instance/webhook/my-form-id`).  
   - Submit a product title and price.  
   - Confirm you are redirected to a valid Stripe payment page.

---

### 5. General Notes & Resources

| Note Content                                                                                       | Context or Link                                                                                          |
|--------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| Workflow created by [n8ninja](https://n8n.ninja).                                                 | Author credit                                                                                           |
| Blog post explaining this workflow in detail: [Automated Stripe Payment Link using n8n](https://www.n8n.ninja/automation-workflow/cGEaQRSS2zo/automated-stripe-payment-link-using-n8n) | Detailed explanation and setup guide                                                                   |
| Video walkthrough on YouTube: [Simplest way to create a Stripe Payment Link](https://youtu.be/cGEaQRSS2zo) | Visual step-by-step guide                                                                                |
| Stripe API documentation integration with n8n credentials: https://docs.n8n.io/integrations/builtin/credentials/stripe/ | For setting up Stripe API credentials in n8n                                                          |
| Important: Price must be in the smallest currency unit (e.g., cents), hence multiplication by 100 | Currency formatting critical for Stripe API compliance                                                 |
| This workflow assumes a single quantity purchase (quantity fixed to 1 in payment link creation)   | Modify if multiple quantities or complex line items needed                                             |

---

This completes the comprehensive reference documentation for the “Simplest way to create a Stripe Payment Link” workflow.