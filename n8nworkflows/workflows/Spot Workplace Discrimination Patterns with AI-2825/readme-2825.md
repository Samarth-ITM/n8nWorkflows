Spot Workplace Discrimination Patterns with AI

https://n8nworkflows.xyz/workflows/spot-workplace-discrimination-patterns-with-ai-2825


# Spot Workplace Discrimination Patterns with AI

---

## 1. Workflow Overview

This workflow, titled **"Spot Workplace Discrimination Patterns with AI"**, is designed to analyze workplace review data from Glassdoor to identify potential patterns of discrimination or bias across demographic groups. It targets HR analysts, diversity officers, researchers, and advocates aiming to quantify workplace disparities using publicly available employee reviews.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and Initialization**  
  Receives manual trigger input and sets the target company name and demographic keys.

- **1.2 Data Acquisition via Web Scraping**  
  Uses ScrapingBee API to scrape Glassdoor search results, company page, and reviews content.

- **1.3 Data Extraction and AI-Powered Parsing**  
  Extracts HTML content for overall review summaries and demographic modules, then uses AI models to extract structured rating data.

- **1.4 Statistical Analysis of Ratings**  
  Calculates variance, standard deviation, z-scores, effect sizes, and p-values (p-scores) to quantify disparities.

- **1.5 Data Formatting and Visualization**  
  Prepares datasets for QuickChart visualizations (scatterplot and bar chart) to highlight discrimination patterns.

- **1.6 AI Textual Analysis and Reporting**  
  Uses OpenAI to generate a textual summary of key takeaways and employee experience insights.

---

## 2. Block-by-Block Analysis

### 2.1 Input Reception and Initialization

**Overview:**  
This block starts the workflow manually, sets the company name for analysis, and defines a dictionary mapping demographic keys to descriptive strings.

**Nodes Involved:**  
- When clicking ‘Test workflow’  
- SET company_name  
- Define dictionary of demographic keys

**Node Details:**

- **When clicking ‘Test workflow’**  
  - Type: Manual Trigger  
  - Role: Entry point for manual execution  
  - Configuration: No parameters  
  - Inputs: None  
  - Outputs: Triggers SET company_name  
  - Edge cases: None

- **SET company_name**  
  - Type: Set  
  - Role: Defines the target company name for scraping  
  - Configuration: Sets string variable `company_name` to "Twilio" (default)  
  - Inputs: From manual trigger  
  - Outputs: To Define dictionary of demographic keys  
  - Edge cases: User must replace default company name for other analyses

- **Define dictionary of demographic keys**  
  - Type: Set  
  - Role: Defines demographic group keys and their descriptive labels used later for visualization labels  
  - Configuration: Assigns multiple string variables such as `asian` = "Asian", `female` = "Women", etc.  
  - Inputs: From SET company_name  
  - Outputs: To ScrapingBee Search Glassdoor  
  - Edge cases: Must include all demographic keys expected in data extraction

---

### 2.2 Data Acquisition via Web Scraping

**Overview:**  
This block scrapes Glassdoor data using ScrapingBee API, navigating from search results to company page to reviews page.

**Nodes Involved:**  
- ScrapingBee Search Glassdoor  
- Extract company url path  
- ScrapingBee GET company page contents  
- Extract reviews page url path  
- ScrapingBee GET Glassdoor Reviews Content

**Node Details:**

- **ScrapingBee Search Glassdoor**  
  - Type: HTTP Request  
  - Role: Queries Glassdoor search results for the company name  
  - Configuration: Sends GET request to ScrapingBee API with query parameter `url` set to Glassdoor search URL for the company name (URL-encoded lowercase)  
  - Credentials: ScrapingBee Query Auth (HTTP Query Auth)  
  - Inputs: From Define dictionary of demographic keys  
  - Outputs: To Extract company url path  
  - Edge cases: Potential scraping failures due to proxy limits, network errors, or Glassdoor page structure changes

- **Extract company url path**  
  - Type: HTML Extract  
  - Role: Extracts the relative URL path to the company page from search results HTML  
  - Configuration: Extracts `href` attribute from CSS selector `body main div a`  
  - Inputs: From ScrapingBee Search Glassdoor  
  - Outputs: To ScrapingBee GET company page contents  
  - Edge cases: Selector may fail if Glassdoor changes page layout

- **ScrapingBee GET company page contents**  
  - Type: HTTP Request  
  - Role: Fetches the company page HTML content  
  - Configuration: Sends GET request to ScrapingBee API with `url` parameter set to full Glassdoor company page URL (base + extracted path)  
  - Credentials: ScrapingBee Query Auth  
  - Inputs: From Extract company url path  
  - Outputs: To Extract reviews page url path  
  - Edge cases: Same as above, plus potential rate limits

- **Extract reviews page url path**  
  - Type: HTML Extract  
  - Role: Extracts the relative URL path to the reviews page from company page HTML  
  - Configuration: Extracts `href` attribute from CSS selector `#reviews a`  
  - Inputs: From ScrapingBee GET company page contents  
  - Outputs: To ScrapingBee GET Glassdoor Reviews Content  
  - Edge cases: Selector fragility

- **ScrapingBee GET Glassdoor Reviews Content**  
  - Type: HTTP Request  
  - Role: Fetches the reviews page HTML content  
  - Configuration: Sends GET request to ScrapingBee API with `url` parameter set to full Glassdoor reviews page URL  
  - Credentials: ScrapingBee Query Auth  
  - Inputs: From Extract reviews page url path  
  - Outputs: To Extract Demographics Module and Extract Overall Review Summary  
  - Edge cases: Same as above

---

### 2.3 Data Extraction and AI-Powered Parsing

**Overview:**  
Extracts relevant HTML sections for overall review summary and demographics, then uses AI models to parse structured rating data.

**Nodes Involved:**  
- Extract Overall Review Summary  
- Extract Demographics Module  
- OpenAI Chat Model2  
- OpenAI Chat Model1  
- Extract overall ratings and distribution percentages  
- Extract demographic distributions

**Node Details:**

- **Extract Overall Review Summary**  
  - Type: HTML Extract  
  - Role: Extracts the overall review summary HTML block from reviews page  
  - Configuration: Extracts inner HTML of `div[data-test="review-summary"]`  
  - Inputs: From ScrapingBee GET Glassdoor Reviews Content  
  - Outputs: To OpenAI Chat Model2  
  - Edge cases: Selector may fail if Glassdoor changes layout

- **Extract Demographics Module**  
  - Type: HTML Extract  
  - Role: Extracts the demographics module HTML block from reviews page  
  - Configuration: Extracts inner HTML of `div[data-test="demographics-module"]`  
  - Inputs: From ScrapingBee GET Glassdoor Reviews Content  
  - Outputs: To OpenAI Chat Model1  
  - Edge cases: Same as above

- **OpenAI Chat Model2**  
  - Type: OpenAI Chat Model (LangChain)  
  - Role: Extracts overall ratings and star distribution percentages from review summary text  
  - Configuration: Uses OpenAI API with default options, input text is the extracted review summary HTML content  
  - Credentials: OpenAI API  
  - Inputs: From Extract Overall Review Summary  
  - Outputs: To Extract overall ratings and distribution percentages (Information Extractor)  
  - Edge cases: API errors, rate limits, unexpected input format

- **OpenAI Chat Model1**  
  - Type: OpenAI Chat Model (LangChain)  
  - Role: Extracts demographic-specific average ratings and review counts from demographics module text  
  - Configuration: Uses OpenAI API with default options, input text is the extracted demographics HTML content  
  - Credentials: OpenAI API  
  - Inputs: From Extract Demographics Module  
  - Outputs: To Extract demographic distributions (Information Extractor)  
  - Edge cases: Same as above

- **Extract overall ratings and distribution percentages**  
  - Type: Information Extractor (LangChain)  
  - Role: Parses structured numeric attributes from the overall review summary text (average rating, total reviews, star distribution percentages)  
  - Configuration: Defines required numeric attributes with descriptions  
  - Inputs: From OpenAI Chat Model2  
  - Outputs: To Define contributions to variance  
  - Edge cases: Extraction failures, missing attributes

- **Extract demographic distributions**  
  - Type: Information Extractor (LangChain)  
  - Role: Parses structured numeric attributes for each demographic group’s average rating and total reviews  
  - Configuration: Defines many numeric attributes for each demographic group (e.g., asian_average_rating, male_total_number_of_reviews, etc.)  
  - Inputs: From OpenAI Chat Model1  
  - Outputs: To Merge node (second input)  
  - Edge cases: Missing data, zero values handled by AI prompt instructions

---

### 2.4 Statistical Analysis of Ratings

**Overview:**  
Calculates statistical measures (variance, standard deviation, z-scores, effect sizes, p-values) to quantify disparities in workplace ratings by demographic groups.

**Nodes Involved:**  
- Define contributions to variance  
- Set variance and std_dev  
- Merge  
- Calculate Z-Scores and Effect Sizes  
- Calculate P-Scores  
- Sort Effect Sizes

**Node Details:**

- **Define contributions to variance**  
  - Type: Set  
  - Role: Calculates contribution to variance for each star rating based on distribution percentages and squared deviations from average rating  
  - Configuration: Uses expressions to compute contribution for 5-star to 1-star ratings  
  - Inputs: From Extract overall ratings and distribution percentages  
  - Outputs: To Set variance and std_dev  
  - Edge cases: Division by zero unlikely; input data must be valid

- **Set variance and std_dev**  
  - Type: Set  
  - Role: Calculates total variance (sum of contributions), standard deviation (sqrt of variance), and carries forward average rating and total reviews  
  - Configuration: Uses expressions summing contributions and computing sqrt for std_dev  
  - Inputs: From Define contributions to variance  
  - Outputs: To Merge  
  - Edge cases: Variance must be non-negative

- **Merge**  
  - Type: Merge (combine by position)  
  - Role: Combines outputs from Set variance and std_dev and Extract demographic distributions into a single JSON object for further analysis  
  - Configuration: Combine mode "combineByPosition"  
  - Inputs: From Set variance and std_dev (main input 0) and Extract demographic distributions (main input 1)  
  - Outputs: To Calculate Z-Scores and Effect Sizes  
  - Edge cases: Mismatched input lengths or missing data

- **Calculate Z-Scores and Effect Sizes**  
  - Type: Set  
  - Role: Calculates z-scores and effect sizes for each demographic group using formulas:  
    - z-score = (group_avg_rating - overall_avg_rating) / (std_dev / sqrt(group_review_count))  
    - effect size = (group_avg_rating - overall_avg_rating) / std_dev  
  - Configuration: Assigns many variables under `population_analysis` object for z_scores, effect_sizes, and review_count  
  - Inputs: From Merge  
  - Outputs: To Calculate P-Scores and Format dataset for scatterplot  
  - Edge cases: Division by zero if review counts are zero; handled by later node

- **Calculate P-Scores**  
  - Type: Code (JavaScript)  
  - Role: Calculates two-tailed p-values for each z-score using approximate standard normal CDF; removes groups with zero reviews from analysis  
  - Configuration: Custom JS code implementing approximate CDF and filtering  
  - Inputs: From Calculate Z-Scores and Effect Sizes  
  - Outputs: To Sort Effect Sizes  
  - Edge cases: Numerical precision, zero or missing review counts

- **Sort Effect Sizes**  
  - Type: Set  
  - Role: Sorts the effect sizes object by ascending values for better visualization and reporting  
  - Configuration: Uses JS expression to sort object entries by effect size value  
  - Inputs: From Calculate P-Scores  
  - Outputs: To QuickChart Bar Chart and Text Analysis of Bias Data  
  - Edge cases: Empty or missing effect sizes

---

### 2.5 Data Formatting and Visualization

**Overview:**  
Formats the statistical results into datasets suitable for QuickChart visualizations and generates scatterplot and bar chart images.

**Nodes Involved:**  
- Format dataset for scatterplot  
- Specify additional parameters for scatterplot  
- Quickchart Scatterplot  
- QuickChart Bar Chart

**Node Details:**

- **Format dataset for scatterplot**  
  - Type: Code (JavaScript)  
  - Role: Converts z-scores, effect sizes, and review counts into a dataset array for scatterplot visualization; includes demographic labels from dictionary  
  - Configuration: Iterates over demographic keys, includes only groups with review count > 0, deletes population_analysis object after formatting  
  - Inputs: From Calculate Z-Scores and Effect Sizes (via Calculate P-Scores)  
  - Outputs: To Specify additional parameters for scatterplot  
  - Edge cases: Missing or zero review counts filtered out

- **Specify additional parameters for scatterplot**  
  - Type: Set  
  - Role: Defines chart type ("scatter") and detailed chart options (title, axis labels, legend, datalabels plugin) for QuickChart API  
  - Configuration: Uses JSON object with dynamic title referencing company name  
  - Inputs: From Format dataset for scatterplot  
  - Outputs: To Quickchart Scatterplot  
  - Edge cases: Chart options must be valid JSON

- **Quickchart Scatterplot**  
  - Type: HTTP Request  
  - Role: Sends chart configuration to QuickChart API to generate scatterplot image URL or binary data  
  - Configuration: Sends GET request with chart config as URL-encoded JSON in query parameter `c`  
  - Inputs: From Specify additional parameters for scatterplot  
  - Outputs: Final scatterplot visualization  
  - Edge cases: API rate limits, network errors

- **QuickChart Bar Chart**  
  - Type: QuickChart Node  
  - Role: Generates a bar chart image showing effect sizes by demographic group  
  - Configuration: Uses effect sizes values and keys as data and labels, sets chart format to PNG, and labels dataset with company name  
  - Inputs: From Sort Effect Sizes  
  - Outputs: Final bar chart visualization  
  - Edge cases: Empty effect sizes object

---

### 2.6 AI Textual Analysis and Reporting

**Overview:**  
Uses OpenAI to generate a textual summary of key takeaways and employee experience insights based on the statistical analysis.

**Nodes Involved:**  
- Text Analysis of Bias Data  
- OpenAI Chat Model

**Node Details:**

- **Text Analysis of Bias Data**  
  - Type: Chain LLM (LangChain)  
  - Role: Sends population analysis data to OpenAI with a prompt requesting 2-5 key takeaways and a section on employee experiences describing workplace perceptions for groups with significantly different experiences  
  - Configuration: Prompt includes instructions and JSON stringified population_analysis data  
  - Inputs: From OpenAI Chat Model (final node after sorting effect sizes)  
  - Outputs: Final textual AI analysis report  
  - Edge cases: API errors, prompt misinterpretation

- **OpenAI Chat Model**  
  - Type: OpenAI Chat Model (LangChain)  
  - Role: Invokes the AI model to process the prompt and generate structured output for Text Analysis of Bias Data  
  - Configuration: Default options, uses OpenAI API credentials  
  - Inputs: From Sort Effect Sizes (parallel branch)  
  - Outputs: To Text Analysis of Bias Data  
  - Edge cases: API limits, latency

---

## 3. Summary Table

| Node Name                          | Node Type                         | Functional Role                                  | Input Node(s)                          | Output Node(s)                          | Sticky Note                                                                                                               |
|-----------------------------------|----------------------------------|-------------------------------------------------|--------------------------------------|---------------------------------------|---------------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’      | Manual Trigger                   | Entry point to start workflow                    | None                                 | SET company_name                      |                                                                                                                           |
| SET company_name                   | Set                              | Defines target company name                       | When clicking ‘Test workflow’         | Define dictionary of demographic keys |                                                                                                                           |
| Define dictionary of demographic keys | Set                              | Defines demographic key-label mappings           | SET company_name                     | ScrapingBee Search Glassdoor          |                                                                                                                           |
| ScrapingBee Search Glassdoor       | HTTP Request                    | Scrapes Glassdoor search results                  | Define dictionary of demographic keys | Extract company url path              | ## Use ScrapingBee to gather raw data from Glassdoor<br>### Due to javascript restrictions, a normal HTTP request cannot be used to gather user-reported details from Glassdoor. Instead, [ScrapingBee](https://www.scrapingbee.com/) offers a great tool with a very generous package of free tokens per month, which works out to roughly 4-5 runs of this workflow. |
| Extract company url path           | HTML Extract                    | Extracts company page URL path                     | ScrapingBee Search Glassdoor         | ScrapingBee GET company page contents |                                                                                                                           |
| ScrapingBee GET company page contents | HTTP Request                    | Fetches company page HTML                          | Extract company url path             | Extract reviews page url path          |                                                                                                                           |
| Extract reviews page url path      | HTML Extract                    | Extracts reviews page URL path                     | ScrapingBee GET company page contents | ScrapingBee GET Glassdoor Reviews Content |                                                                                                                           |
| ScrapingBee GET Glassdoor Reviews Content | HTTP Request                    | Fetches reviews page HTML                          | Extract reviews page url path        | Extract Demographics Module, Extract Overall Review Summary |                                                                                                                           |
| Extract Overall Review Summary     | HTML Extract                    | Extracts overall review summary HTML               | ScrapingBee GET Glassdoor Reviews Content | OpenAI Chat Model2                   |                                                                                                                           |
| Extract Demographics Module        | HTML Extract                    | Extracts demographics module HTML                  | ScrapingBee GET Glassdoor Reviews Content | OpenAI Chat Model1                   |                                                                                                                           |
| OpenAI Chat Model2                 | OpenAI Chat Model (LangChain)  | Processes overall review summary for ratings      | Extract Overall Review Summary       | Extract overall ratings and distribution percentages |                                                                                                                           |
| OpenAI Chat Model1                 | OpenAI Chat Model (LangChain)  | Processes demographics module for demographic ratings | Extract Demographics Module          | Extract demographic distributions     |                                                                                                                           |
| Extract overall ratings and distribution percentages | Information Extractor (LangChain) | Extracts numeric overall ratings and star distributions | OpenAI Chat Model2                   | Define contributions to variance       |                                                                                                                           |
| Extract demographic distributions | Information Extractor (LangChain) | Extracts numeric demographic ratings and counts   | OpenAI Chat Model1                   | Merge (input 1)                      |                                                                                                                           |
| Define contributions to variance  | Set                              | Calculates variance contributions from star ratings | Extract overall ratings and distribution percentages | Set variance and std_dev             | ### Calculate variance and standard deviation from review rating distributions.                                           |
| Set variance and std_dev          | Set                              | Calculates total variance and standard deviation  | Define contributions to variance     | Merge (input 0)                      |                                                                                                                           |
| Merge                            | Merge                            | Combines variance/std_dev data with demographic data | Set variance and std_dev, Extract demographic distributions | Calculate Z-Scores and Effect Sizes |                                                                                                                           |
| Calculate Z-Scores and Effect Sizes | Set                              | Calculates z-scores, effect sizes, and review counts | Merge                             | Calculate P-Scores, Format dataset for scatterplot |                                                                                                                           |
| Calculate P-Scores               | Code                             | Calculates p-values from z-scores and filters groups with zero reviews | Calculate Z-Scores and Effect Sizes | Sort Effect Sizes                    |                                                                                                                           |
| Sort Effect Sizes                | Set                              | Sorts effect sizes ascending for visualization     | Calculate P-Scores                  | QuickChart Bar Chart, OpenAI Chat Model |                                                                                                                           |
| QuickChart Bar Chart             | QuickChart                       | Generates bar chart visualization of effect sizes | Sort Effect Sizes                  | None                                |                                                                                                                           |
| Format dataset for scatterplot  | Code                             | Formats data for scatterplot visualization         | Calculate Z-Scores and Effect Sizes | Specify additional parameters for scatterplot | ## Formatting datasets for Scatterplot                                                                                   |
| Specify additional parameters for scatterplot | Set                              | Defines scatterplot chart options and labels       | Format dataset for scatterplot      | Quickchart Scatterplot              |                                                                                                                           |
| Quickchart Scatterplot          | HTTP Request                    | Generates scatterplot image via QuickChart API     | Specify additional parameters for scatterplot | None                                |                                                                                                                           |
| OpenAI Chat Model               | OpenAI Chat Model (LangChain)  | Invokes AI for textual analysis of bias data       | Sort Effect Sizes                  | Text Analysis of Bias Data          |                                                                                                                           |
| Text Analysis of Bias Data      | Chain LLM (LangChain)           | Generates AI summary and insights from population analysis | OpenAI Chat Model                 | None                                |                                                                                                                           |

---

## 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - Purpose: Start workflow manually

2. **Create Set Node "SET company_name"**  
   - Type: Set  
   - Assign variable `company_name` with default value `"Twilio"` (string)  
   - Connect from Manual Trigger

3. **Create Set Node "Define dictionary of demographic keys"**  
   - Type: Set  
   - Assign multiple string variables mapping demographic keys to descriptive labels, e.g.:  
     - `asian` = "Asian"  
     - `female` = "Women"  
     - `trans` = "Transgender and/or Non-Binary"  
     - ... (include all keys as per original)  
   - Connect from "SET company_name"

4. **Create HTTP Request Node "ScrapingBee Search Glassdoor"**  
   - Type: HTTP Request  
   - Method: GET  
   - URL: `https://app.scrapingbee.com/api/v1`  
   - Authentication: HTTP Query Auth (ScrapingBee credentials)  
   - Query Parameters:  
     - `url` = `https://www.glassdoor.com/Search/results.htm?keyword={{ $json.company_name.toLowerCase().urlEncode() }}`  
     - `premium_proxy` = `true`  
     - `block_resources` = `false`  
     - `stealth_proxy` = `true`  
   - Connect from "Define dictionary of demographic keys"

5. **Create HTML Extract Node "Extract company url path"**  
   - Type: HTML Extract  
   - Operation: Extract HTML content  
   - Extraction: CSS selector `body main div a`, attribute `href`  
   - Output key: `url_path`  
   - Connect from "ScrapingBee Search Glassdoor"

6. **Create HTTP Request Node "ScrapingBee GET company page contents"**  
   - Same base URL and auth as step 4  
   - Query Parameter `url` = `https://www.glassdoor.com{{ $json.url_path }}`  
   - Connect from "Extract company url path"

7. **Create HTML Extract Node "Extract reviews page url path"**  
   - CSS selector `#reviews a`, attribute `href`  
   - Output key: `url_path`  
   - Connect from "ScrapingBee GET company page contents"

8. **Create HTTP Request Node "ScrapingBee GET Glassdoor Reviews Content"**  
   - Same base URL and auth as step 4  
   - Query Parameter `url` = `https://www.glassdoor.com{{ $json.url_path }}`  
   - Connect from "Extract reviews page url path"

9. **Create HTML Extract Node "Extract Overall Review Summary"**  
   - CSS selector `div[data-test="review-summary"]`  
   - Return inner HTML as `review_summary`  
   - Connect from "ScrapingBee GET Glassdoor Reviews Content"

10. **Create HTML Extract Node "Extract Demographics Module"**  
    - CSS selector `div[data-test="demographics-module"]`  
    - Return inner HTML as `demographics_content`  
    - Connect from "ScrapingBee GET Glassdoor Reviews Content"

11. **Create OpenAI Chat Model Node "OpenAI Chat Model2"**  
    - Use OpenAI API credentials  
    - Input text: `={{ $json.review_summary }}`  
    - Connect from "Extract Overall Review Summary"

12. **Create OpenAI Chat Model Node "OpenAI Chat Model1"**  
    - Use OpenAI API credentials  
    - Input text: `={{ $json.demographics_content }}`  
    - Connect from "Extract Demographics Module"

13. **Create Information Extractor Node "Extract overall ratings and distribution percentages"**  
    - Input text: `={{ $json.review_summary }}`  
    - Define numeric attributes: average_rating, total_number_of_reviews, 5_star_distribution_percentage, ..., 1_star_distribution_percentage (all required)  
    - Connect from "OpenAI Chat Model2"

14. **Create Information Extractor Node "Extract demographic distributions"**  
    - Input text: `={{ $json.demographics_content }}`  
    - Define numeric attributes for each demographic group’s average rating and total reviews (e.g., asian_average_rating, asian_total_number_of_reviews, etc.)  
    - Connect from "OpenAI Chat Model1"

15. **Create Set Node "Define contributions to variance"**  
    - Calculate contribution_to_variance for each star rating using formula:  
      `(star_distribution_percentage / 100) * (star_value - average_rating)^2`  
    - Connect from "Extract overall ratings and distribution percentages"

16. **Create Set Node "Set variance and std_dev"**  
    - Calculate variance as sum of contributions  
    - Calculate std_dev as sqrt(variance)  
    - Pass average_rating and total_number_of_reviews forward  
    - Connect from "Define contributions to variance"

17. **Create Merge Node "Merge"**  
    - Mode: Combine by position  
    - Input 0: From "Set variance and std_dev"  
    - Input 1: From "Extract demographic distributions"

18. **Create Set Node "Calculate Z-Scores and Effect Sizes"**  
    - For each demographic group:  
      - z_score = (group_avg_rating - average_rating) / (std_dev / sqrt(group_review_count))  
      - effect_size = (group_avg_rating - average_rating) / std_dev  
      - Store review counts  
    - Connect from "Merge"

19. **Create Code Node "Calculate P-Scores"**  
    - Implement approximate normal CDF function  
    - Calculate p_scores = 2 * normSDist(-abs(z_score)) for each group with review_count > 0  
    - Remove groups with zero review counts from z_scores, effect_sizes, p_scores  
    - Connect from "Calculate Z-Scores and Effect Sizes"

20. **Create Set Node "Sort Effect Sizes"**  
    - Sort population_analysis.effect_sizes by ascending value  
    - Connect from "Calculate P-Scores"

21. **Create QuickChart Node "QuickChart Bar Chart"**  
    - Data: effect_sizes values  
    - Labels: effect_sizes keys  
    - Chart type: bar chart, PNG format  
    - Dataset label: `{{ $json.company_name }} Effect Size on Employee Experience`  
    - Connect from "Sort Effect Sizes"

22. **Create Code Node "Format dataset for scatterplot"**  
    - Format z_scores, effect_sizes, and review_count into dataset array for scatterplot  
    - Include demographic labels from dictionary  
    - Remove population_analysis from output  
    - Connect from "Calculate Z-Scores and Effect Sizes"

23. **Create Set Node "Specify additional parameters for scatterplot"**  
    - Set chart type to "scatter"  
    - Define chart options: title (dynamic with company name), axis labels, legend, datalabels plugin styling  
    - Connect from "Format dataset for scatterplot"

24. **Create HTTP Request Node "Quickchart Scatterplot"**  
    - URL: `https://quickchart.io/chart`  
    - Query parameter `c`: JSON stringified chart config from previous node  
    - Connect from "Specify additional parameters for scatterplot"

25. **Create OpenAI Chat Model Node "OpenAI Chat Model"**  
    - Use OpenAI API credentials  
    - Connect from "Sort Effect Sizes"

26. **Create Chain LLM Node "Text Analysis of Bias Data"**  
    - Input text: Prompt with instructions plus population_analysis JSON stringified data  
    - Connect from "OpenAI Chat Model"

---

## 5. General Notes & Resources

| Note Content                                                                                                                        | Context or Link                                                                                   |
|------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Spot Workplace Discrimination Patterns using ScrapingBee, Glassdoor, OpenAI, and QuickChart                                        | Workflow title and main technology stack                                                        |
| Due to javascript restrictions, a normal HTTP request cannot be used to gather user-reported details from Glassdoor. Instead, ScrapingBee offers a great tool with a very generous package of free tokens per month, which works out to roughly 4-5 runs of this workflow. | [ScrapingBee](https://www.scrapingbee.com/)                                                     |
| Inspired by [Wes Medford's Medium Post](https://medium.com/@wryanmedford/an-open-letter-to-twilios-leadership-f06f661ecfb4)        | Original data analysis inspiration and credit                                                   |
| Glossary of statistical terms: Z-Score, Effect Size, P-Score (P-Value) and their relevance to workplace equity analysis           | Sticky note explaining statistical concepts used                                                |
| Example AI Analysis (Twilio Example) highlighting significant disparities among disabled employees, LGBTQIA community, transgender employees, veterans, and gender discrepancies | Sticky note with detailed interpretation and key takeaways                                     |
| Estimated setup time: ~20 minutes. Replace ScrapingBee and OpenAI credentials, input company name, run workflow, review outputs.  | User instructions                                                                              |
| Example visualizations include scatterplots and bar charts generated by QuickChart API                                             | Visual output examples shown in sticky notes                                                    |

---

# End of Document