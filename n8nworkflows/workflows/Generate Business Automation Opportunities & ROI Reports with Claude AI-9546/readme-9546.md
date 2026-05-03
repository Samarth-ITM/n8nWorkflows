Generate Business Automation Opportunities & ROI Reports with Claude AI

https://n8nworkflows.xyz/workflows/generate-business-automation-opportunities---roi-reports-with-claude-ai-9546


# Generate Business Automation Opportunities & ROI Reports with Claude AI

---

## 1. Workflow Overview

This workflow enables a personalized business automation opportunity audit powered by Claude AI (Anthropic). It targets small to medium businesses seeking to identify high-impact automation use cases with estimated ROI. The process collects detailed business input through a web form, then sequentially applies four AI agents specialized in business analysis, process mapping, automation design, and ROI calculation. Finally, it formats and emails a comprehensive, conversion-focused report to the user.

**Target Use Cases:**  
- Businesses wanting to discover automation potentials  
- Consultants providing tailored automation audits  
- Sales funnels for automation service offerings  

**Logical Blocks:**  
- **1.1 Input Reception:** Captures detailed business info via a form.  
- **1.2 Variable Initialization:** Extracts and organizes form data into variables for AI processing.  
- **1.3 AI Processing Pipeline:** Sequential execution of four AI agents, each building on the previous outputs:  
  - Agent 1: Business Analyst (deep business model and operational insights)  
  - Agent 2: Process Mapper (maps workflows, bottlenecks, and time wastes)  
  - Agent 3: Automation Architect (designs specific automation opportunities)  
  - Agent 4: ROI Calculator (calculates financial ROI and ranks opportunities)  
- **1.4 Email Report Formatting:** Compiles all AI outputs into a rich HTML email with business branding and CTAs.  
- **1.5 Email Delivery:** Sends the personalized report to the user’s email address.  

---

## 2. Block-by-Block Analysis

### 1.1 Input Reception

**Overview:**  
Captures detailed business information through a custom web form to fuel AI analysis.

**Nodes Involved:**  
- Form Submission

**Node Details:**  

- **Form Submission**  
  - Type: Form Trigger (Webhook-based)  
  - Role: Entry point capturing user input via a business audit form titled "Free Automation Opportunity Audit - By Evervise."  
  - Configuration:  
    - 9 fields including business description, industry, team size (dropdown), repetitive tasks, current tools, bottlenecks, revenue range (optional), email, and name.  
    - Required fields: business description, team size, repetitive tasks, email, name.  
    - Form description emphasizes discovering hidden time savings and ROI-ranked automation opportunities.  
  - Input: Web form data submitted by user  
  - Output: JSON with form field values  
  - Failure Modes:  
    - Incomplete submission if required fields omitted (frontend validation recommended)  
    - Possible webhook misconfiguration causing no trigger  
  - Version: 2.3

---

### 1.2 Variable Initialization

**Overview:**  
Extracts and normalizes form data into workflow variables for consistent use downstream.

**Nodes Involved:**  
- Initialize Variables

**Node Details:**  

- **Initialize Variables**  
  - Type: Set Node  
  - Role: Assigns form fields to named variables with fallback defaults for optional fields.  
  - Configuration:  
    - Maps each form field JSON value to readable variable names (e.g., `business_description`, `industry`, `team_size`, etc.)  
    - Uses fallback values like 'General', 'Not specified', or 'Not disclosed' for empty optional inputs.  
    - Adds a timestamp variable with ISO string of current date-time.  
  - Input: JSON from Form Submission  
  - Output: JSON with organized variables for AI nodes  
  - Failure Modes:  
    - Expression errors if form fields missing or renamed  
  - Version: 3.4

---

### 1.3 AI Processing Pipeline

**Overview:**  
Runs four specialized AI agents in sequence, each using Claude Sonnet 4.5 model with tailored prompts and settings, to generate a detailed automation audit.

**Nodes Involved:**  
- Business Analyst Model  
- Agent 1: Business Analyst  
- Process Mapper Model  
- Agent 2: Process Mapper  
- Automation Architect Model  
- Agent 3: Automation Architect  
- ROI Calculator Model  
- Agent 4: ROI Calculator

**Node Details:**  

- **Business Analyst Model**  
  - Type: LangChain Anthropic Chat Completion  
  - Role: AI model node for business analysis prompt  
  - Config: Claude Sonnet 4.5, temperature 0.4, topP 0.9  
  - Input: Business variables from Initialize Variables  
  - Output: Business analysis text  
  - Credentials: Anthropic API  
  - Failure Modes: API errors, rate limits, prompt timeout  

- **Agent 1: Business Analyst**  
  - Type: LangChain Agent  
  - Role: Wraps Business Analyst Model with a detailed system prompt and user prompt dynamically embedding variables (business description, industry, etc.)  
  - Configuration: Requires understanding of business nuances, outputs structured analysis (overview, insights, workflows, pain points, tech stack, scalability, industry context)  
  - Input: Initialized variables  
  - Output: Business analysis JSON with key insights  
  - Failure Modes: Expression or prompt rendering errors, API failures  

- **Process Mapper Model**  
  - Type: LangChain Anthropic Chat Completion  
  - Role: Process mapping AI model with temperature 0.5, topP 0.9  
  - Input: Business Analyst output + form fields related to tasks, bottlenecks, tools  
  - Output: Text mapping processes, bottlenecks, time waste, automation potential  

- **Agent 2: Process Mapper**  
  - Type: LangChain Agent  
  - Role: Uses Process Mapper Model with a system prompt focused on comprehensive process mapping, including hidden processes and time calculations  
  - Input: Output from Agent 1 + form variables  
  - Output: Structured process map analysis  
  - Failure Modes: Similar to Agent 1  

- **Automation Architect Model**  
  - Type: LangChain Anthropic Chat Completion  
  - Role: AI model generating automation solution designs, temperature 0.6, topP 0.95  
  - Input: Business analysis + process map + current tools  
  - Output: Automation opportunity designs with complexity, tools, benefits  

- **Agent 3: Automation Architect**  
  - Type: LangChain Agent  
  - Role: Wraps Automation Architect Model with a prompt to create 15+ automation ideas, priority matrix, tech stack suggestions  
  - Input: Output from Agent 1 and Agent 2 + tools list  
  - Output: Detailed automation architecture text  
  - Failure Modes: Prompt complexity, API limits  

- **ROI Calculator Model**  
  - Type: LangChain Anthropic Chat Completion  
  - Role: Calculates financial ROI metrics for automations, temperature 0.3, topP 0.85  
  - Input: Team size, revenue, industry, process map, automation solutions  
  - Output: Ranked list of top 10 automations by ROI, implementation roadmap  

- **Agent 4: ROI Calculator**  
  - Type: LangChain Agent  
  - Role: Wraps ROI Calculator Model focused on financial and strategic analysis, prioritization, scaling impact  
  - Input: Outputs from prior agents + variables  
  - Output: ROI report text and roadmap  
  - Failure Modes: Financial estimate inaccuracies, prompt parsing errors  

**General AI Failure Modes:** Network/API errors, prompt malformations, rate limiting, unexpected input formats, model response inconsistencies.

---

### 1.4 Email Report Formatting

**Overview:**  
Formats all AI outputs into a polished, branded HTML email report with embedded metrics, CTAs, FAQ, and testimonials designed for high conversion.

**Nodes Involved:**  
- Format Email Report

**Node Details:**  

- **Format Email Report**  
  - Type: Code Node (JavaScript)  
  - Role:  
    - Extracts key financial metrics (total savings, ROI, payback period) from Agent 4 output via regex  
    - Combines outputs from all agents into styled HTML sections with headings and icons  
    - Embeds personalized user data (name, industry, team size)  
    - Includes testimonial, pricing tiers, FAQ, and CTAs with booking links  
    - Uses responsive CSS for mobile-friendly display  
  - Input: JSON outputs from all agents + variables  
  - Output: JSON with `email_html`, `email_subject`, `user_email`, etc. for sending  
  - Failure Modes: Parsing errors if AI output format changes, regex misses, code exceptions  

---

### 1.5 Email Delivery

**Overview:**  
Sends the formatted email report to the user’s provided email address using Gmail credentials.

**Nodes Involved:**  
- Send Email Report

**Node Details:**  

- **Send Email Report**  
  - Type: Gmail Node  
  - Role: Sends the HTML email report with subject and recipient from previous node’s output  
  - Configuration:  
    - Uses OAuth2 Gmail credentials  
    - Sends to user’s email from form input  
    - Subject line includes user name and savings amount  
  - Input: From Format Email Report node  
  - Output: Email sent confirmation  
  - Failure Modes: Authentication errors, invalid email address, Gmail API limits, network issues  

---

## 3. Summary Table

| Node Name                | Node Type                         | Functional Role                      | Input Node(s)            | Output Node(s)              | Sticky Note                                                                                      |
|--------------------------|----------------------------------|------------------------------------|--------------------------|----------------------------|------------------------------------------------------------------------------------------------|
| Form Submission          | Form Trigger                     | Capture business info from user    | -                        | Initialize Variables        | ## 📝 FORM INTAKE Captures detailed business info with 9 fields for quality analysis           |
| Initialize Variables     | Set                             | Normalize and assign variables     | Form Submission          | Agent 1: Business Analyst   |                                                                                                |
| Business Analyst Model   | LangChain Anthropic Chat Model  | Business model deep analysis       | Initialize Variables      | Agent 1: Business Analyst   |                                                                                                |
| Agent 1: Business Analyst| LangChain Agent                 | Wraps Business Analyst Model       | Initialize Variables      | Agent 2: Process Mapper     | ## 🤖 AGENT 1: Business Analyst Deep business insights, pain points, workflows, scalability    |
| Process Mapper Model     | LangChain Anthropic Chat Model  | Map business processes and bottlenecks | Agent 1: Business Analyst | Agent 2: Process Mapper     |                                                                                                |
| Agent 2: Process Mapper  | LangChain Agent                 | Wraps Process Mapper Model         | Agent 1: Business Analyst | Agent 3: Automation Architect | ## 🤖 AGENT 2: Process Mapper Maps workflows, time waste, bottlenecks, quick wins               |
| Automation Architect Model| LangChain Anthropic Chat Model | Design automation solutions        | Agent 2: Process Mapper   | Agent 3: Automation Architect |                                                                                                |
| Agent 3: Automation Architect| LangChain Agent             | Wraps Automation Architect Model   | Agent 2: Process Mapper   | Agent 4: ROI Calculator     | ## 🤖 AGENT 3: Automation Architect Designs 15+ automations, priority matrix, real tools        |
| ROI Calculator Model     | LangChain Anthropic Chat Model  | Calculate ROI and rank automations | Agent 3: Automation Architect | Agent 4: ROI Calculator  |                                                                                                |
| Agent 4: ROI Calculator  | LangChain Agent                 | Wraps ROI Calculator Model         | Agent 3: Automation Architect | Format Email Report        | ## 🤖 AGENT 4: ROI Calculator Calculates time & cost savings, payback, ROI, roadmap            |
| Format Email Report      | Code                            | Format all AI outputs into email   | Agent 4: ROI Calculator  | Send Email Report           | ## 📧 EMAIL REPORT Stunning HTML report with savings banner, CTAs, testimonial, pricing tiers   |
| Send Email Report        | Gmail                           | Send report email to user          | Format Email Report       | -                          |                                                                                                |
| Workflow Overview       | Sticky Note                     | Overview and stats summary         | -                        | -                          | ## 📊 WORKFLOW OVERVIEW AI pipeline, conversion funnel, expected conversion stats              |
| Agent 1 Info            | Sticky Note                     | Agent 1 prompt and goals summary   | -                        | -                          |                                                                                                |
| Agent 2 Info            | Sticky Note                     | Agent 2 prompt and goals summary   | -                        | -                          |                                                                                                |
| Agent 3 Info            | Sticky Note                     | Agent 3 prompt and goals summary   | -                        | -                          |                                                                                                |
| Agent 4 Info            | Sticky Note                     | Agent 4 prompt and goals summary   | -                        | -                          |                                                                                                |
| Form Section            | Sticky Note                     | Form field description and tips    | -                        | -                          |                                                                                                |
| Email Section           | Sticky Note                     | Email report features and design   | -                        | -                          |                                                                                                |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a "Form Trigger" node: "Form Submission"**  
   - Configure webhook ID (e.g., `automation-finder-form-001`)  
   - Title: "Free Automation Opportunity Audit - By Evervise"  
   - Add 9 form fields:  
     - Textarea: "What does your business do?" (required)  
     - Text: "Industry/Niche" (optional)  
     - Dropdown: "Team size" with options ["Just me (solo)", "2-5 people", "6-10 people", "11-25 people", "26-50 people", "50+ people"] (required)  
     - Textarea: "What repetitive tasks take most of your team's time?" (required)  
     - Textarea: "Current tools/software you use" (optional)  
     - Textarea: "Biggest frustration or bottleneck" (optional)  
     - Dropdown: "Monthly revenue range (optional - helps with ROI estimates)" with specified options  
     - Email: "Your Email" (required)  
     - Text: "Your Name" (required)  
   - Add form description about discovering hidden savings and automation opportunities.

3. **Add a "Set" node: "Initialize Variables"**  
   - Connect from Form Submission  
   - Assign variables from form fields with fallback values for optional fields:  
     - `business_description` ← "What does your business do?"  
     - `industry` ← "Industry/Niche" or 'General'  
     - `team_size` ← "Team size"  
     - `repetitive_tasks` ← "What repetitive tasks take most..."  
     - `current_tools` ← "Current tools/software you use" or 'Not specified'  
     - `biggest_bottleneck` ← "Biggest frustration or bottleneck" or 'Not specified'  
     - `monthly_revenue` ← "Monthly revenue range" or 'Not disclosed'  
     - `user_email` ← "Your Email"  
     - `user_name` ← "Your Name"  
     - `analysis_timestamp` ← current ISO datetime string

4. **Add four LangChain Anthropic Chat Completion nodes:**  
   - **"Business Analyst Model"**: model "claude-sonnet-4-5-20250929", temperature 0.4, topP 0.9  
   - **"Process Mapper Model"**: same model, temperature 0.5, topP 0.9  
   - **"Automation Architect Model"**: same model, temperature 0.6, topP 0.95  
   - **"ROI Calculator Model"**: same model, temperature 0.3, topP 0.85  
   - Connect each model node as input for its respective agent node (next step).

5. **Add four LangChain Agent nodes wrapping each model:**  

   - **Agent 1: Business Analyst**  
     - Connect from Initialize Variables  
     - Use Business Analyst Model as languageModel  
     - System prompt: expert business analyst instructions (see overview)  
     - User prompt: includes business info variables interpolated from Initialize Variables node  

   - **Agent 2: Process Mapper**  
     - Connect from Agent 1  
     - Use Process Mapper Model  
     - System prompt: process mapping expert instructions  
     - User prompt: includes Agent 1 output and relevant variables (repetitive tasks, bottleneck, current tools)  

   - **Agent 3: Automation Architect**  
     - Connect from Agent 2  
     - Use Automation Architect Model  
     - System prompt: automation architect instructions  
     - User prompt: includes Agent 1 output, Agent 2 output, current tools  

   - **Agent 4: ROI Calculator**  
     - Connect from Agent 3  
     - Use ROI Calculator Model  
     - System prompt: financial analyst and ROI specialist instructions  
     - User prompt: includes team size, revenue, industry, Agent 2 output, Agent 3 output  

6. **Connect Agent 4 output to a "Code" node: "Format Email Report"**  
   - Implement JavaScript to parse key financial metrics from Agent 4 output via regex  
   - Combine all agents’ outputs and user variables into a styled HTML email template with sections, CTA buttons, testimonial, pricing tiers, FAQ, and footer  
   - Output JSON: `email_html`, `email_subject`, `user_email`, `user_name`, `total_savings`, `payback_period`, `roi_percentage`, `timestamp`

7. **Add a Gmail node: "Send Email Report"**  
   - Connect from Format Email Report node  
   - Configure Gmail OAuth2 credentials  
   - Set recipient to `={{ $json.user_email }}`  
   - Set subject to `={{ $json.email_subject }}`  
   - Set message to `={{ $json.email_html }}`  
   - Enable HTML content sending  

8. **Optional: Add Sticky Notes** to document workflow overview, form section, and agent descriptions for maintainability and team knowledge sharing.

9. **Test the entire flow** by submitting the form, verifying AI node outputs, email formatting, and email delivery.

---

## 5. General Notes & Resources

| Note Content                                                                                                             | Context or Link                                         |
|--------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------|
| Workflow completion time averages 50-70 seconds with four AI agents in sequence.                                          | Sticky Note: Workflow Overview                          |
| Cost per analysis estimated between $0.20 and $0.30 using Anthropic Claude Sonnet 4.5.                                   | Sticky Note: Workflow Overview                          |
| Conversion funnel tiers: Free blueprint, $497 for top 3 automations built, $1,997 for all 10 automations plus support.    | Sticky Note: Workflow Overview                          |
| Email report designed to maximize conversion with dynamic savings banner, social proof, FAQ, pricing tiers, and CTAs.     | Sticky Note: Email Section                              |
| AI prompts carefully structured for each agent to maximize insight quality and relevance.                                | Sticky Notes: Agent 1-4 Info                            |
| ROI calculations use conservative rates based on team size and industry, building trust with transparent math.           | Sticky Note: Agent 4 Info                               |
| The form is optimized to balance detail needed for quality AI analysis and user friction minimization.                   | Sticky Note: Form Section                               |
| Booking links in CTAs direct users to Evervise’s strategy call and service pages.                                         | Embedded in email HTML and CTA buttons                  |

---

**Disclaimer:**  
The provided description and workflow content is derived exclusively from an n8n automation workflow. It complies fully with content policies and contains no illegal, offensive, or protected material. All data handled is legal and publicly accessible.

---