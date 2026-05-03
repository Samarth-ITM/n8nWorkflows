Custom Deal Recommendations by Email using Forms, Bright Data & GPT-4o-mini

https://n8nworkflows.xyz/workflows/custom-deal-recommendations-by-email-using-forms--bright-data---gpt-4o-mini-3304


# Custom Deal Recommendations by Email using Forms, Bright Data & GPT-4o-mini

### 1. Workflow Overview

This n8n workflow automates the delivery of personalized "Top Deals of the Day" from MediaMarkt to users via email. It is designed for e-commerce deal enthusiasts who want curated product recommendations based on their selected categories.

The workflow is logically divided into the following blocks:

- **1.1 User Input Collection:** Captures user preferences (product categories) and email via a web form.
- **1.2 Web Scraping of Deals:** Uses Bright Data to scrape MediaMarkt’s deals page and retrieve raw HTML content.
- **1.3 HTML Content Extraction:** Parses the scraped HTML to extract relevant page elements (title and body).
- **1.4 AI-based Deal Recommendation:** Sends extracted content and user preferences to GPT-4o-mini to generate a filtered, categorized list of recommended deals.
- **1.5 Data Processing & Formatting:** Splits the AI-generated JSON list into individual deal items and generates a structured HTML email using a template.
- **1.6 Email Delivery & User Feedback:** Sends the personalized deals email to the user and displays a confirmation page.

---

### 2. Block-by-Block Analysis

#### 2.1 User Input Collection

- **Overview:**  
  This block triggers the workflow when a user submits their preferences via a form. It collects selected product categories and the user’s email address.

- **Nodes Involved:**  
  - When User Completes Form

- **Node Details:**

  - **When User Completes Form**  
    - Type: Form Trigger  
    - Role: Entry point; listens for form submissions at webhook path `/get-top-deals`.  
    - Configuration:  
      - Form title: "Top deals"  
      - Fields:  
        - Category (dropdown, multiselect, required) with options like Appliances, Cameras, Cell Phones, etc.  
        - Email (email input, required)  
      - Button label: "Get Deals"  
      - Response mode: returns output of last node in workflow to the form page.  
      - Ignores bots to prevent spam.  
    - Inputs: HTTP form submission  
    - Outputs: JSON containing user-selected categories and email.  
    - Edge cases:  
      - Missing required fields (form validation handles this).  
      - Bot submissions ignored.  
      - Invalid email format rejected by form.  
    - Version: 2.2

---

#### 2.2 Web Scraping of Deals

- **Overview:**  
  Scrapes MediaMarkt’s deals page using Bright Data proxy-based scraping to bypass restrictions and obtain the raw HTML content.

- **Nodes Involved:**  
  - Get MediaMarkt Offers Website

- **Node Details:**

  - **Get MediaMarkt Offers Website**  
    - Type: Bright Data (community node)  
    - Role: Fetches the deals page HTML content from MediaMarkt Spain site.  
    - Configuration:  
      - URL: `https://www.mediamarkt.es/es/campaign/campanas-y-ofertas`  
      - Zone: `web_unlocker1` (Bright Data proxy zone)  
      - Country: Spain (`es`)  
      - Format: JSON (Bright Data returns structured data)  
    - Credentials: Bright Data API key required.  
    - Inputs: Triggered by form submission node.  
    - Outputs: JSON containing scraped page data.  
    - Edge cases:  
      - Bright Data API key invalid or expired → authentication error.  
      - Network timeout or proxy failure.  
      - Changes in MediaMarkt page structure may affect scraping results.  
    - Version: 1

---

#### 2.3 HTML Content Extraction

- **Overview:**  
  Extracts the page title and body HTML from the scraped data for further processing.

- **Nodes Involved:**  
  - Extract Body and Title from Website

- **Node Details:**

  - **Extract Body and Title from Website**  
    - Type: HTML Extract  
    - Role: Parses the scraped HTML content to extract the `<title>` and `<body>` elements.  
    - Configuration:  
      - Operation: Extract HTML content  
      - Extraction keys:  
        - `title` from CSS selector `title`  
        - `body` from CSS selector `body`  
      - Trims extracted values.  
    - Inputs: Output from Bright Data node.  
    - Outputs: JSON with `title` and `body` properties containing extracted HTML/text.  
    - Edge cases:  
      - If scraped HTML is malformed or empty, extraction may fail or return empty strings.  
      - Changes in page structure could affect selectors.  
    - Version: 1.2

---

#### 2.4 AI-based Deal Recommendation

- **Overview:**  
  Sends the extracted HTML body and user-selected categories to OpenAI GPT-4o-mini to generate a JSON list of recommended deals, classified by category.

- **Nodes Involved:**  
  - Generate List of Deals by Category

- **Node Details:**

  - **Generate List of Deals by Category**  
    - Type: OpenAI (LangChain node)  
    - Role: Uses GPT-4o-mini to analyze the scraped content and user preferences, then generate a filtered and categorized list of deals.  
    - Configuration:  
      - Model: `gpt-4o-mini`  
      - Messages:  
        - System prompt instructs to generate a JSON list with properties: name, description, price, link, category inside a `results` property. Also requests translation to English if needed.  
        - User prompt includes:  
          - The extracted HTML body text.  
          - The categories selected by the user, joined as a comma-separated string.  
      - JSON output enabled to parse AI response as JSON.  
    - Credentials: OpenAI API key required.  
    - Inputs: Extracted HTML body and user categories.  
    - Outputs: JSON with `results` array containing deal objects.  
    - Edge cases:  
      - OpenAI API key invalid or quota exceeded → authentication or rate limit errors.  
      - AI response malformed or not JSON → parsing errors.  
      - Input text too long → truncation or API errors.  
    - Version: 1.8

---

#### 2.5 Data Processing & Formatting

- **Overview:**  
  Splits the AI-generated JSON list into individual deal items and generates a structured HTML email using a template.

- **Nodes Involved:**  
  - Extract items from results  
  - Create HTML for Email

- **Node Details:**

  - **Extract items from results**  
    - Type: Split Out  
    - Role: Splits the `results` array from the AI output into individual items for templating.  
    - Configuration:  
      - Field to split out: `message.content.results` (path to deals array in AI response)  
    - Inputs: AI-generated JSON with deals list.  
    - Outputs: Individual deal items as separate JSON objects.  
    - Edge cases:  
      - If `results` is missing or empty, node outputs nothing.  
      - Malformed AI output → errors or empty output.  
    - Version: 1

  - **Create HTML for Email**  
    - Type: Document Generator (community node)  
    - Role: Uses a Handlebars template to generate an HTML snippet listing the recommended deals.  
    - Configuration:  
      - Template:  
        ```
        <br>
        These are our recommended deals today:<br>
        <ul>
        {{#each items}}
        <li>{{category}}: <a href="https://www.bestbuy.com{{link}}">{{name}}</a> for {{price}}€</li>
        {{/each}}
        </ul>
        <br>
        ```  
      - One template for all items (aggregated).  
    - Inputs: Individual deal items from Split Out node.  
    - Outputs: HTML string with formatted deals list.  
    - Edge cases:  
      - If no items, email body will be empty or incomplete.  
      - Template assumes links are relative to BestBuy domain (may be incorrect for MediaMarkt).  
    - Version: 1

---

#### 2.6 Email Delivery & User Feedback

- **Overview:**  
  Sends the generated HTML email to the user’s email address and displays a confirmation page on the form.

- **Nodes Involved:**  
  - Notify End User by Email  
  - Show Form Results Page

- **Node Details:**

  - **Notify End User by Email**  
    - Type: Email Send  
    - Role: Sends the personalized deals email via SMTP.  
    - Configuration:  
      - Subject: "Your last deals!"  
      - To: User’s email from form submission (`When User Completes Form` node JSON)  
      - From: `deals@n8nhackers.com`  
      - HTML body:  
        ```
        Hi!<br>
        {{ $json.text }}
        Best,<br>
        The n8nhackers team!
        ```  
      - No additional SMTP options configured.  
    - Credentials: SMTP account required.  
    - Inputs: HTML content from Document Generator node; user email from form node.  
    - Outputs: Confirmation of email sent.  
    - Edge cases:  
      - SMTP authentication failure or network issues.  
      - Invalid user email address (though form validation should prevent this).  
      - Email content empty if previous nodes fail.  
    - Version: 2.1

  - **Show Form Results Page**  
    - Type: Form Completion  
    - Role: Displays a confirmation message on the form page after email is sent.  
    - Configuration:  
      - Completion title: "Our recommended deals!"  
      - Completion message:  
        ```
        We have sent {{ $('Extract items from results').all().length }} recommended deals to your email!
        ```  
      - Operation: Completion page display.  
    - Inputs: Triggered after email node completes.  
    - Outputs: HTTP response to user’s browser.  
    - Edge cases:  
      - If no deals found, message count will be zero.  
      - If previous nodes fail, user may see misleading confirmation.  
    - Version: 1

---

### 3. Summary Table

| Node Name                     | Node Type                          | Functional Role                      | Input Node(s)                | Output Node(s)               | Sticky Note                                                                                      |
|-------------------------------|----------------------------------|------------------------------------|-----------------------------|-----------------------------|------------------------------------------------------------------------------------------------|
| When User Completes Form       | Form Trigger                     | Collect user preferences and email | (Webhook HTTP)               | Get MediaMarkt Offers Website |                                                                                                |
| Get MediaMarkt Offers Website  | Bright Data (community node)     | Scrape MediaMarkt deals page       | When User Completes Form     | Extract Body and Title from Website | Requires Bright Data API key; community node only works on self-hosted n8n.                    |
| Extract Body and Title from Website | HTML Extract                   | Extract page title and body HTML   | Get MediaMarkt Offers Website | Generate List of Deals by Category |                                                                                                |
| Generate List of Deals by Category | OpenAI (LangChain)              | Generate AI-filtered deal list     | Extract Body and Title from Website | Extract items from results     | Requires OpenAI API key; uses GPT-4o-mini model.                                               |
| Extract items from results     | Split Out                       | Split AI JSON list into items      | Generate List of Deals by Category | Create HTML for Email          |                                                                                                |
| Create HTML for Email          | Document Generator (community node) | Generate HTML email content         | Extract items from results   | Notify End User by Email      | Community node only works on self-hosted n8n; template can be customized.                      |
| Notify End User by Email       | Email Send                      | Send personalized deals email      | Create HTML for Email        | Show Form Results Page        | Requires SMTP credentials; sends email to user email from form.                               |
| Show Form Results Page         | Form Completion                 | Display confirmation page          | Notify End User by Email     | (HTTP response to user)       |                                                                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Form Trigger Node**  
   - Type: Form Trigger  
   - Set webhook path to `/get-top-deals`  
   - Configure form title: "Top deals"  
   - Add form fields:  
     - Category (Dropdown, multiselect, required) with options: Appliances, Cameras, Car Electronics, Cell Phones, Computers & Tablets, TV & Home Theater, Video Games  
     - Email (Email input, required)  
   - Set button label: "Get Deals"  
   - Enable "Ignore Bots" option  
   - Set response mode to "Last Node"  

2. **Add Bright Data Node**  
   - Type: Bright Data (community node)  
   - Set URL: `https://www.mediamarkt.es/es/campaign/campanas-y-ofertas`  
   - Select zone: `web_unlocker1`  
   - Select country: Spain (`es`)  
   - Set format to JSON  
   - Configure Bright Data API credentials (API key)  

3. **Add HTML Extract Node**  
   - Type: HTML Extract  
   - Operation: Extract HTML content  
   - Extraction keys:  
     - `title` with CSS selector `title`  
     - `body` with CSS selector `body`  
   - Enable trimming of values  

4. **Add OpenAI Node (LangChain)**  
   - Type: OpenAI (LangChain)  
   - Model: `gpt-4o-mini`  
   - Messages:  
     - System: Instruct to generate JSON list of deals with properties: name, description, price, link, category inside `results`. Translate to English if needed.  
     - User: Pass extracted HTML body text.  
     - User: Pass user-selected categories joined by commas.  
   - Enable JSON output  
   - Configure OpenAI API credentials  

5. **Add Split Out Node**  
   - Type: Split Out  
   - Field to split out: `message.content.results` (path to deals array in AI response)  

6. **Add Document Generator Node**  
   - Type: Document Generator (community node)  
   - Template:  
     ```
     <br>
     These are our recommended deals today:<br>
     <ul>
     {{#each items}}
     <li>{{category}}: <a href="https://www.bestbuy.com{{link}}">{{name}}</a> for {{price}}€</li>
     {{/each}}
     </ul>
     <br>
     ```  
   - Set "One Template" to true  
   - Note: Customize HTML/CSS as needed  
   - Requires Document Generator community node installed  

7. **Add Email Send Node**  
   - Type: Email Send  
   - Subject: "Your last deals!"  
   - To: Expression referencing user email from form node: `={{ $('When User Completes Form').first().json.Email}}`  
   - From: `deals@n8nhackers.com`  
   - HTML body:  
     ```
     Hi!<br>
     {{ $json.text }}
     Best,<br>
     The n8nhackers team!
     ```  
   - Configure SMTP credentials for sending email  

8. **Add Form Completion Node**  
   - Type: Form Completion  
   - Completion title: "Our recommended deals!"  
   - Completion message:  
     ```
     We have sent {{ $('Extract items from results').all().length }} recommended deals to your email!
     ```  

9. **Connect Nodes in Order:**  
   - When User Completes Form → Get MediaMarkt Offers Website  
   - Get MediaMarkt Offers Website → Extract Body and Title from Website  
   - Extract Body and Title from Website → Generate List of Deals by Category  
   - Generate List of Deals by Category → Extract items from results  
   - Extract items from results → Create HTML for Email  
   - Create HTML for Email → Notify End User by Email  
   - Notify End User by Email → Show Form Results Page  

10. **Test Workflow:**  
    - Submit the form with test categories and email.  
    - Verify email receipt with personalized deals.  
    - Adjust template or prompts as needed.  

---

### 5. General Notes & Resources

| Note Content                                                                                                  | Context or Link                                                                                   |
|---------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| This workflow uses community nodes Bright Data and Document Generator, which require self-hosted n8n instances. | Bright Data: https://www.npmjs.com/package/n8n-nodes-brightdata                                  |
|                                                                                                               | Document Generator: https://www.npmjs.com/package/n8n-nodes-document-generator                   |
| Workflow template and nodes created by Miquel Colomer and n8nhackers.com                                      | Miquel Colomer LinkedIn: https://www.linkedin.com/in/miquelcolomersalas/                         |
|                                                                                                               | n8nhackers website: https://n8nhackers.com                                                      |
| For consulting and support, contact: contact@n8nhackers.com                                                  |                                                                                                |
| The email template uses relative links pointing to BestBuy domain, which may be incorrect for MediaMarkt deals. | Consider updating links to correct MediaMarkt URLs in the template.                             |
| You can extend the form to include price ranges or brands, or schedule the workflow with Cron for automation. |                                                                                                |

---

This documentation provides a detailed, structured understanding of the workflow, enabling reproduction, modification, and troubleshooting by advanced users or AI agents.