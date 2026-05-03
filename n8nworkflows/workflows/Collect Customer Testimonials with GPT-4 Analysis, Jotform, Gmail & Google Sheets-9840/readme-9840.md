Collect Customer Testimonials with GPT-4 Analysis, Jotform, Gmail & Google Sheets

https://n8nworkflows.xyz/workflows/collect-customer-testimonials-with-gpt-4-analysis--jotform--gmail---google-sheets-9840


# Collect Customer Testimonials with GPT-4 Analysis, Jotform, Gmail & Google Sheets

### 1. Workflow Overview

This workflow automates the collection, analysis, and utilization of customer testimonials submitted via Jotform. It centralizes testimonial reception, applies AI-powered analysis (using GPT-4) to extract marketing insights, logs detailed and analyzed testimonial data into Google Sheets, and automates customer engagement through personalized thank-you emails with coupon incentives. The workflow also notifies the marketing team with summarized insights to facilitate rapid campaign integration.

**Target Use Cases:**  
- Businesses seeking to streamline and scale testimonial collection.  
- Marketing teams needing ready-to-use testimonial content optimized for various channels.  
- Customer engagement teams automating appreciation and incentives.  

**Logical Blocks:**  
- **1.1 Input Reception:** Capture testimonial submissions from Jotform.  
- **1.2 Data Extraction:** Parse and structure raw testimonial form data.  
- **1.3 AI Processing:** Analyze testimonial text for tone, sentiment, marketing elements using GPT-4.  
- **1.4 Data Logging:** Store raw and analyzed testimonial data in Google Sheets.  
- **1.5 Customer Engagement:** Generate and send personalized thank-you emails with coupons.  
- **1.6 Marketing Notification:** Inform marketing team with key insights and testimonial alerts.  

---

### 2. Block-by-Block Analysis

---

#### 1.1 Input Reception

- **Overview:**  
  Initiates the workflow by triggering on new testimonial submissions via Jotform.

- **Nodes Involved:**  
  - Jotform Trigger

- **Node Details:**  
  - **Jotform Trigger**  
    - *Type:* Trigger node specialized for Jotform form submissions.  
    - *Configuration:* Connected to a specific Jotform (form ID `252892971568071`) to listen for new entries.  
    - *Key Parameters:* Uses Jotform API credentials.  
    - *Inputs:* External form submission webhook.  
    - *Outputs:* Passes the full JSON submission data downstream.  
    - *Edge Cases:* Possible webhook failures, API connectivity issues, or invalid form data. Requires proper Jotform API credentials and active webhook.  
    - *Version:* n8n built-in Jotform Trigger v1.

---

#### 1.2 Data Extraction

- **Overview:**  
  Extracts and normalizes key testimonial fields from raw form data for downstream processing.

- **Nodes Involved:**  
  - Extract Testimonial Data

- **Node Details:**  
  - **Extract Testimonial Data**  
    - *Type:* Set node that maps raw JSON fields into standardized variables.  
    - *Configuration:* Assigns variables such as customerName, customerEmail, companyName, productService, testimonialText, rating, and adds a timestamp (submittedAt).  
    - *Key Expressions:* Uses expressions like `={{ $json['Customer Name'] }}` and `$now.toISO()`.  
    - *Inputs:* Output from Jotform Trigger.  
    - *Outputs:* Structured JSON object with cleaned data fields.  
    - *Edge Cases:* Missing or malformed fields in form data can cause empty or incorrect assignments. Validate form field presence upstream.  
    - *Version:* Set node v3.3.

---

#### 1.3 AI Processing

- **Overview:**  
  Uses GPT-4 based LangChain agent to analyze testimonial text and extract marketing insights including sentiment, tone, best quotes, and marketing copy variants.

- **Nodes Involved:**  
  - OpenAI Chat Model  
  - AI Testimonial Analysis  
  - Parse AI Analysis

- **Node Details:**  
  - **OpenAI Chat Model**  
    - *Type:* Language model node using OpenAI GPT-4.1-mini model variant.  
    - *Configuration:* Prepares input text for AI analysis; no additional options specified.  
    - *Credentials:* Uses OpenAI API credentials.  
    - *Inputs:* Receives structured data from Extract Testimonial Data indirectly through AI Testimonial Analysis.  
    - *Outputs:* Raw AI model response forwarded to AI Testimonial Analysis.  
    - *Edge Cases:* API rate limits, network failures, unexpected AI outputs.  
    - *Version:* LangChain LmChatOpenAi v1.2.

  - **AI Testimonial Analysis**  
    - *Type:* LangChain Agent node configured as conversational agent.  
    - *Configuration:* Receives testimonial details and prompts the agent to extract a predefined JSON object with fields like overallTone, sentimentScore, emotionalImpact, authenticity, bestQuote, alternateQuotes, keyBenefitsMentioned, painPointsAddressed, specificResults, marketingUseCase, targetAudience, twitterVersion, linkedinVersion, websiteVersion, headlineVersion, calloutWords, marketingPriority.  
    - *System Message:* Sets the assistant role as a marketing copywriter focused on extracting compelling marketing elements.  
    - *Inputs:* Data from Extract Testimonial Data and OpenAI Chat Model.  
    - *Outputs:* JSON-formatted AI analysis results.  
    - *Edge Cases:* Incorrect or malformed JSON output from AI, timeout or API errors. Requires validation of AI response format downstream.  
    - *Version:* LangChain agent v1.6.

  - **Parse AI Analysis**  
    - *Type:* Set node that extracts the AI response string from the AI Testimonial Analysis output for further use.  
    - *Configuration:* Maps the `output` field from AI JSON response to `output` variable.  
    - *Inputs:* AI Testimonial Analysis node output.  
    - *Outputs:* Passes parsed AI analysis JSON string forward.  
    - *Edge Cases:* If AI output is missing or malformed, parsing will fail or produce invalid downstream data.  
    - *Version:* Set node v3.3.

---

#### 1.4 Data Logging

- **Overview:**  
  Logs detailed testimonial data and AI analysis results into a Google Sheets document for centralized, searchable testimonial management.

- **Nodes Involved:**  
  - Log to Testimonial Library

- **Node Details:**  
  - **Log to Testimonial Library**  
    - *Type:* Google Sheets node using appendOrUpdate operation.  
    - *Configuration:* Maps testimonial fields (email, rating, company name, customer name, product/service, testimonial text) to specific columns in Sheet2 of a designated Google Sheets document (ID: `1K0lqkb-z0fbdb_pEUDtUfl5N2jQ_nT-YEQtylwB4968`).  
    - *Inputs:* Data from Parse AI Analysis (but mapping uses original Jotform Trigger data).  
    - *Outputs:* None (terminal side effect).  
    - *Credentials:* Requires Google Sheets OAuth2 credentials.  
    - *Edge Cases:* Sheets API quota limits, permission errors, column mapping mismatches, or sheet not found. Validate Sheet ID and OAuth scopes before running.  
    - *Version:* Google Sheets node v4.4.

---

#### 1.5 Customer Engagement

- **Overview:**  
  Generates personalized coupon codes based on testimonial rating, then sends a customized thank-you email to the customer including the coupon and referral incentives.

- **Nodes Involved:**  
  - Generate Coupon Code  
  - Send Thank You Email

- **Node Details:**  
  - **Generate Coupon Code**  
    - *Type:* Code node running JavaScript.  
    - *Configuration:* Creates a coupon code by combining "THANKS", first 5 letters of customer name, and a random number. Adjusts discount percent according to rating (20% for 5 stars, 15% for 4 stars, 10% default). Sets a 30-day expiry date.  
    - *Inputs:* Testimonial data from Parse AI Analysis (actually using Jotform submission data).  
    - *Outputs:* JSON with couponCode, discountPercent, expiryDays, expiryDate, customerName, customerEmail, rating.  
    - *Edge Cases:* Missing customer name or rating could produce invalid coupon codes or defaults.  
    - *Version:* Code node v2.

  - **Send Thank You Email**  
    - *Type:* Gmail node configured to send emails via OAuth2.  
    - *Configuration:* Sends email to customer's email with subject "Thank You for Your Testimonial!" and message body containing coupon code, discount percent, and expiry date placeholders (currently hardcoded text placeholders "CODE", "DISCOUNT", "EXPIRY" need to be programmatically replaced for final use).  
    - *Inputs:* Output of Generate Coupon Code.  
    - *Outputs:* None.  
    - *Credentials:* Gmail OAuth2 credentials required.  
    - *Edge Cases:* Email sending failures, invalid email addresses, OAuth token expiration.  
    - *Version:* Gmail node v2.1.

---

#### 1.6 Marketing Notification

- **Overview:**  
  Sends an email notification to the marketing team alerting them of a new testimonial, including AI analysis summary and usage recommendations for rapid campaign integration.

- **Nodes Involved:**  
  - Notify Marketing Team

- **Node Details:**  
  - **Notify Marketing Team**  
    - *Type:* Gmail node for outbound email.  
    - *Configuration:* Sends email to marketing@yourcompany.com with subject "New Testimonial Received" and a brief message prompting them to check the testimonial library.  
    - *Inputs:* Output from Generate Coupon Code (after customer email trigger).  
    - *Outputs:* None.  
    - *Credentials:* Gmail OAuth2 credentials needed.  
    - *Edge Cases:* Email delivery failures, invalid marketing email configured.  
    - *Version:* Gmail node v2.1.

---

### 3. Summary Table

| Node Name               | Node Type                      | Functional Role                           | Input Node(s)             | Output Node(s)                      | Sticky Note                                                                                                                                                           |
|-------------------------|--------------------------------|-----------------------------------------|---------------------------|-----------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Sticky Note             | Sticky Note                    | Informational overview                   | -                         | -                                 | Customer Testimonial Collector - Purpose: Transform scattered testimonials into organized marketing assets with AI-powered analysis. Key Features and Impact noted. |
| Sticky Note1            | Sticky Note                    | Jotform field details                     | -                         | -                                 | Jotform Testimonial Fields: Customer Name, Customer Email, Company Name, Job Title, Product or Service Used, Testimonial Text, Rating, Permission to Share, Photo Upload, Use Case or Industry. Create your form for free on [Jotform using this link](https://www.jotform.com/?partner=mediajade) |
| Jotform Trigger         | Jotform Trigger                | Testimonial form submission trigger      | -                         | Extract Testimonial Data           |                                                                                                                                                                     |
| Extract Testimonial Data | Set                           | Normalize form data                       | Jotform Trigger           | AI Testimonial Analysis            |                                                                                                                                                                     |
| Sticky Note2            | Sticky Note                    | AI analysis overview                      | -                         | -                                 | AI Testimonial Analysis - AI Analyzes For: Tone and Sentiment, Best Quote Extraction, Key Benefits Mentioned, Emotional Impact Score, Marketing Use Cases, Social Media Optimization. Output: Marketing-ready assets |
| OpenAI Chat Model       | LangChain OpenAI LLM           | Underlying GPT-4 model                    | AI Testimonial Analysis   | AI Testimonial Analysis            |                                                                                                                                                                     |
| AI Testimonial Analysis | LangChain Agent                | Extract marketing insights from testimonial | Extract Testimonial Data  | Parse AI Analysis                  |                                                                                                                                                                     |
| Parse AI Analysis       | Set                           | Extract AI output string                  | AI Testimonial Analysis   | Log to Testimonial Library, Generate Coupon Code |                                                                                                                                                                     |
| Sticky Note3            | Sticky Note                    | Google Sheets testimonial database overview | -                       | -                                 | Testimonial Content Log - Google Sheets Database: Complete testimonial details, AI analysis and quotes, Marketing versions, Permission tracking, Searchable library. Enables: Easy marketing access |
| Log to Testimonial Library | Google Sheets                | Append/update testimonial data            | Parse AI Analysis         | Generate Coupon Code               |                                                                                                                                                                     |
| Generate Coupon Code    | Code                          | Create personalized coupon code and discount | Parse AI Analysis       | Send Thank You Email, Notify Marketing Team |                                                                                                                                                                     |
| Sticky Note4            | Sticky Note                    | Automated customer thank you email overview | -                       | -                                 | Customer Thank You - Automated Email: Thank customer, Show usage preview, Exclusive coupon code, Referral incentive, Social share versions. Drives: Loyalty and advocacy |
| Send Thank You Email    | Gmail                         | Send thank-you email with coupon          | Generate Coupon Code      | -                                 |                                                                                                                                                                     |
| Sticky Note5            | Sticky Note                    | Marketing team alert overview             | -                         | -                                 | Marketing Team Alert - Notification Includes: New testimonial available, AI analysis summary, Best quotes, Priority level, Usage recommendations. Enables: Immediate campaign use |
| Notify Marketing Team   | Gmail                         | Notify marketing team of new testimonial  | Generate Coupon Code      | -                                 |                                                                                                                                                                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Jotform for Testimonial Collection**  
   - Fields: Customer Name, Customer Email, Company Name, Job Title, Product/Service Used, Testimonial Text, Rating, Permission to Share, Photo Upload, Use Case or Industry.  
   - Obtain form ID after publishing.

2. **Add Jotform Trigger Node**  
   - Type: Jotform Trigger  
   - Configure with your Jotform API credentials.  
   - Select the created form by form ID.  
   - Set webhook to listen to new submissions.

3. **Add Set Node “Extract Testimonial Data”**  
   - Connect input from Jotform Trigger.  
   - Map form fields to normalized variables: customerName, customerEmail, companyName, productService, testimonialText, rating, submittedAt (current timestamp).  
   - Use expressions like `={{ $json['Customer Name'] }}`.  

4. **Add LangChain Agent Node “AI Testimonial Analysis”**  
   - Connect input from “Extract Testimonial Data”.  
   - Configure system message: “You are a marketing copywriter. Extract compelling elements from testimonials and optimize for marketing channels.”  
   - Set prompt to analyze testimonial and output structured JSON with all required marketing insight fields (tone, sentimentScore, emotionalImpact, best quotes, marketing versions, etc.).  
   - Use an OpenAI GPT-4 model agent.  
   - Ensure OpenAI API credentials are set.

5. **Add Set Node “Parse AI Analysis”**  
   - Connect input from “AI Testimonial Analysis”.  
   - Extract the AI analysis JSON output string into a variable for downstream use.

6. **Add Google Sheets Node “Log to Testimonial Library”**  
   - Connect input from “Parse AI Analysis”.  
   - Configure Google Sheets OAuth2 credentials.  
   - Use appendOrUpdate operation on your testimonial library sheet (Sheet2).  
   - Map testimonial fields and AI data into columns: customer email, rating, company name, customer name, product/service, testimonial text, and AI analysis outputs.  
   - Verify Sheet ID and permission scopes.

7. **Add Code Node “Generate Coupon Code”**  
   - Connect input from “Parse AI Analysis”.  
   - Implement JavaScript to generate a coupon code based on customer name and rating.  
   - Calculate discount percent (20% for 5 stars, 15% for 4 stars, else 10%).  
   - Set expiry date 30 days from submission.  
   - Pass coupon details and customer info downstream.

8. **Add Gmail Node “Send Thank You Email”**  
   - Connect input from “Generate Coupon Code”.  
   - Configure Gmail OAuth2 credentials.  
   - Set recipient to customerEmail.  
   - Compose email with subject “Thank You for Your Testimonial!”.  
   - Construct message body embedding coupon code, discount percent, and expiry date dynamically (replace placeholders).  
   - Confirm email sending options.

9. **Add Gmail Node “Notify Marketing Team”**  
   - Connect input from “Generate Coupon Code”.  
   - Configure to send email to marketing@yourcompany.com.  
   - Subject: “New Testimonial Received”.  
   - Message: Summary notification with direction to testimonial library.  
   - Use Gmail OAuth2 credentials.

10. **Add Sticky Notes Throughout**  
    - Add descriptive sticky notes as per the provided content to document workflow purposes, field mappings, AI analysis scope, database structure, customer engagement strategy, and marketing alert details.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                          | Context or Link                                                                                          |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| Create your form for free on [Jotform using this link](https://www.jotform.com/?partner=mediajade)                                                                                     | Jotform form creation and affiliate link                                                                |
| Customer Testimonial Collector Purpose and Impact: Centralizes testimonials, AI-powered analysis, marketing-ready outputs, 500% increase in testimonial collection.                     | Workflow overview sticky note                                                                            |
| AI Testimonial Analysis extracts tone, sentiment, best quotes, emotional impact, marketing use cases, and social media optimizations producing marketing-ready assets.                   | AI analysis sticky note                                                                                   |
| Google Sheets acts as a searchable testimonial content log including AI analysis results and permission tracking for marketing access.                                                | Data logging sticky note                                                                                  |
| Automated thank-you emails include exclusive coupon codes based on rating and encourage referrals and social sharing, driving loyalty and advocacy.                                   | Customer thank you sticky note                                                                            |
| Marketing Team Alert email enables immediate campaign use by notifying new testimonials with AI summaries and priority recommendations.                                              | Marketing notification sticky note                                                                        |

---

**Disclaimer:** The text provided comes exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.