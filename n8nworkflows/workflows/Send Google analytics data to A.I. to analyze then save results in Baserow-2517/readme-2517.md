Send Google analytics data to A.I. to analyze then save results in Baserow

https://n8nworkflows.xyz/workflows/send-google-analytics-data-to-a-i--to-analyze-then-save-results-in-baserow-2517


# Send Google analytics data to A.I. to analyze then save results in Baserow

### 1. Workflow Overview

This n8n workflow automates the process of extracting Google Analytics data, analyzing it with an AI service, and saving the analyzed insights into a Baserow database for SEO reporting. It is designed for website owners who want to monitor key SEO metrics such as page engagement, country views, and Google Search Console performance without hiring an SEO expert.

The workflow is logically divided into these main blocks:

- **1.1 Trigger Block**: Receives either a scheduled weekly trigger or manual trigger to start the process.
- **1.2 Google Analytics Data Retrieval Block**: Fetches three categories of Google Analytics data for the current week and the prior week:
  - Page engagement stats
  - Google Search Console organic search results
  - Country views data
- **1.3 Data Parsing Block**: Transforms raw Google Analytics responses into simplified, URL-encoded JSON strings suitable for AI consumption.
- **1.4 AI Analysis Block**: Sends the transformed data to Openrouter.ai (an AI service) for SEO analysis and suggestions, comparing the two weeks of data.
- **1.5 Data Storage Block**: Saves the AI-generated reports into a Baserow database, organizing insights into predefined columns.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger Block

- **Overview:** This block initiates the workflow either via a scheduled weekly trigger or manual trigger for testing.
- **Nodes Involved:** 
  - Schedule Trigger
  - When clicking ‘Test workflow’

##### Node Details:

- **Schedule Trigger**
  - Type: Schedule Trigger
  - Role: Automatically runs the workflow every week.
  - Configuration: Interval set to 1 week.
  - Inputs: None
  - Outputs: Connects to “Get Page Engagement Stats for this week”
  - Failures: Cron misconfiguration or n8n scheduler issues can prevent execution.

- **When clicking ‘Test workflow’**
  - Type: Manual Trigger
  - Role: Allows manual execution during workflow testing.
  - Configuration: Default manual trigger.
  - Inputs: None
  - Outputs: Connects to “Get Page Engagement Stats for this week”
  - Failures: None typical.

---

#### 2.2 Google Analytics Data Retrieval Block

- **Overview:** Collects Google Analytics data for three categories (page engagement, search results, country views), each for the current week and prior week, using the Google Analytics OAuth2 credentials and property ID.
- **Nodes Involved:** 
  - Get Page Engagement Stats for this week
  - Get Page Engagement Stats for prior week
  - Get Google Search Results for this week
  - Get Google Search Results for last week
  - Get Country views data for this week
  - Get Country views data for last week

##### Node Details:

- **Get Page Engagement Stats for this week**
  - Type: Google Analytics (GA4)
  - Role: Fetches metrics like screenPageViews, activeUsers, screenPageViewsPerUser, eventCount by page (unifiedScreenName dimension) for the last 7 days.
  - Configuration: propertyId set via credential reference; returnAll enabled to fetch all data.
  - Input: Trigger node
  - Output: To “Parse data from Google Analytics”
  - Failures: GA API auth errors, API quota limits, property ID errors.

- **Get Page Engagement Stats for prior week**
  - Type: Google Analytics (GA4)
  - Role: Same metrics and dimensions as above, but for the 7-day period ending 7 days before today (week before last).
  - Configuration: Custom date range using expressions.
  - Input: From “Parse data from Google Analytics”
  - Output: To “Parse GA data”

- **Get Google Search Results for this week**
  - Type: Google Analytics (GA4)
  - Role: Retrieves organic search-related metrics (activeUsers, engagedSessions, engagementRate, organicGoogleSearchAveragePosition, etc.) by landing page + query string for the last 7 days.
  - Configuration: propertyId with OAuth2 credentials, returnAll true.
  - Input: From “Parse GA data”
  - Output: To “Parse Google Analytics Data”

- **Get Google Search Results for last week**
  - Type: Google Analytics (GA4)
  - Role: Same metrics and dimension as above but for prior week (7-14 days ago).
  - Configuration: Custom date range expressions.
  - Input: From “Parse Google Analytics Data”
  - Output: To “Parse Google Analytics Data1”

- **Get Country views data for this week**
  - Type: Google Analytics (GA4)
  - Role: Fetches country-level metrics (activeUsers, newUsers, engagementRate, sessions, etc.) for the last 7 days.
  - Configuration: propertyId with OAuth2 credentials.
  - Input: From “Parse Google Analytics Data1”
  - Output: To “Parse Google analytics data”

- **Get Country views data for last week**
  - Type: Google Analytics (GA4)
  - Role: Same metrics as above for the prior week (7-14 days ago).
  - Configuration: Custom date range expressions.
  - Input: From “Parse Google analytics data”
  - Output: To “Parse Google analytics data1”

- **Common Failures:**
  - Authentication failures with Google Analytics OAuth2.
  - API quota exceeded.
  - Invalid property ID or permissions issues.
  - Network timeouts or API errors.

---

#### 2.3 Data Parsing Block

- **Overview:** Converts complex Google Analytics query results into simplified, URL-encoded JSON strings that summarize key metrics. This makes data easier to pass into AI analysis nodes.
- **Nodes Involved:** 
  - Parse data from Google Analytics
  - Parse GA data
  - Parse Google Analytics Data
  - Parse Google Analytics Data1
  - Parse Google analytics data
  - Parse Google analytics data1

##### Node Details:

- All these nodes are **Code** nodes with custom JavaScript to:
  - Validate input data structure (checking for array presence, rows existence).
  - Map each row’s dimension and metric values into simplified objects with readable keys (e.g., page, pageViews, activeUsers, country, etc.).
  - Convert the array of objects into JSON, then encode it for URL transmission.
  - Return the encoded JSON string as `urlString` in output.

- Inputs: From respective Google Analytics nodes.
- Outputs: Feed into subsequent Google Analytics nodes or AI HTTP Request nodes.
- Failures:
  - Data shape inconsistencies or missing expected fields cause script errors.
  - Empty or malformed API responses.
  - Runtime exceptions in JavaScript code.

---

#### 2.4 AI Analysis Block

- **Overview:** Sends the parsed Google Analytics data to Openrouter.ai’s chat completion API to receive SEO analysis comparing current week to prior week data. The prompts request markdown tables and SEO improvement suggestions.
- **Nodes Involved:** 
  - Send page data to A.I.
  - Send page Search data to A.I.
  - Send country view data to A.I.

##### Node Details:

- Each node is an **HTTP Request** node configured as:
  - POST request to `https://openrouter.ai/api/v1/chat/completions`
  - JSON body includes:
    - Model: `"meta-llama/llama-3.1-70b-instruct:free"`
    - Messages array with user role content embedding two weeks’ data URL strings from parsed nodes.
    - Prompts instruct the AI to compare data, create markdown tables, and provide 5 SEO suggestions.
  - Authentication: HTTP Header Auth with header `Authorization: Bearer {API_KEY}` (configured as generic credential).
  - Inputs: Parsed data URL strings from parsing nodes.
  - Outputs: Pass results to Baserow node.
- Failures:
  - API key issues (invalid or missing token).
  - Network or timeout errors.
  - API quota or rate limits.
  - Unexpected response format or empty AI output.

---

#### 2.5 Data Storage Block

- **Overview:** Stores the AI-generated SEO reports in a Baserow database table for easy access and tracking.
- **Nodes Involved:** 
  - Save A.I. output to Baserow

##### Node Details:

- Type: Baserow node (create operation).
- Configuration:
  - Database ID and Table ID set.
  - Fields mapped:
    - Name: fixed string “Name of your blog”
    - Country Views: AI output from “Send page data to A.I.”
    - Page Views: AI output from “Send page Search data to A.I.”
    - Search Report: AI output from “Send country view data to A.I.”
    - Blog: current date/time as timestamp.
  - Credentials: Baserow API key or OAuth credentials.
- Inputs: AI response JSON objects.
- Outputs: None (end node).
- Failures:
  - Baserow API authentication errors.
  - Missing or misconfigured table/field IDs.
  - Network or API rate limits.

---

### 3. Summary Table

| Node Name                          | Node Type                 | Functional Role                    | Input Node(s)                      | Output Node(s)                     | Sticky Note                                                                                                                                                                                                                                                         |
|-----------------------------------|---------------------------|----------------------------------|----------------------------------|----------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Schedule Trigger                  | Schedule Trigger          | Starts workflow weekly           | None                             | Get Page Engagement Stats for this week |                                                                                                                                                                                                                                                                     |
| When clicking ‘Test workflow’     | Manual Trigger            | Manual start for testing         | None                             | Get Page Engagement Stats for this week |                                                                                                                                                                                                                                                                     |
| Sticky Note                      | Sticky Note               | Workflow description             | None                             | None                             | ## Send Google analytics to A.I. and save results to baserow This workflow will check for country views, page engagement and google search console results. It will take this week's data and compare it to last week's data. [You can read more about this workflow here](https://rumjahn.com/how-i-used-a-i-to-be-an-seo-expert-and-analyzed-my-google-analytics-data-in-n8n-and-make-com/) |
| Sticky Note1                     | Sticky Note               | Instructions for Google Analytics property ID | None                             | None                             | ## Property ID 1. Create your [Google Analytics Credentials](https://docs.n8n.io/integrations/builtin/credentials/google/oauth-single-service/?utm_source=n8n_app&utm_medium=credential_settings&utm_campaign=create_new_credentials_modal) 2. Enter your [property ID](https://developers.google.com/analytics/devguides/reporting/data/v1/property-id). |
| Get Page Engagement Stats for this week | Google Analytics (GA4)     | Fetch this week’s page engagement data | Schedule Trigger, Manual Trigger | Parse data from Google Analytics |                                                                                                                                                                                                                                                                     |
| Parse data from Google Analytics  | Code                      | Parse this week’s engagement data | Get Page Engagement Stats for this week | Get Page Engagement Stats for prior week |                                                                                                                                                                                                                                                                     |
| Get Page Engagement Stats for prior week | Google Analytics (GA4)     | Fetch prior week’s page engagement data | Parse data from Google Analytics | Parse GA data                   |                                                                                                                                                                                                                                                                     |
| Parse GA data                   | Code                      | Parse prior week’s engagement data | Get Page Engagement Stats for prior week | Get Google Search Results for this week |                                                                                                                                                                                                                                                                     |
| Get Google Search Results for this week | Google Analytics (GA4)     | Fetch this week’s Google search metrics | Parse GA data                   | Parse Google Analytics Data      |                                                                                                                                                                                                                                                                     |
| Parse Google Analytics Data      | Code                      | Parse this week’s Google search data | Get Google Search Results for this week | Get Google Search Results for last week |                                                                                                                                                                                                                                                                     |
| Get Google Search Results for last week | Google Analytics (GA4)     | Fetch prior week’s Google search metrics | Parse Google Analytics Data      | Parse Google Analytics Data1     |                                                                                                                                                                                                                                                                     |
| Parse Google Analytics Data1     | Code                      | Parse prior week’s Google search data | Get Google Search Results for last week | Get Country views data for this week |                                                                                                                                                                                                                                                                     |
| Get Country views data for this week | Google Analytics (GA4)     | Fetch this week’s country-level data | Parse Google Analytics Data1     | Parse Google analytics data      |                                                                                                                                                                                                                                                                     |
| Parse Google analytics data      | Code                      | Parse this week’s country views data | Get Country views data for this week | Get Country views data for last week |                                                                                                                                                                                                                                                                     |
| Get Country views data for last week | Google Analytics (GA4)     | Fetch prior week’s country-level data | Parse Google analytics data      | Parse Google analytics data1     |                                                                                                                                                                                                                                                                     |
| Parse Google analytics data1     | Code                      | Parse prior week’s country views data | Get Country views data for last week | Send page data to A.I.           |                                                                                                                                                                                                                                                                     |
| Send page data to A.I.           | HTTP Request              | Send page engagement data to AI  | Parse Google analytics data1     | Send page Search data to A.I.    | ## Send data to A.I. Fill in your Openrouter A.I. credentials. Use Header Auth. - Username: Authorization - Password: Bearer {insert your API key} Remember to add a space after bearer. Also, feel free to modify the prompt to A.1.                                    |
| Send page Search data to A.I.    | HTTP Request              | Send search console data to AI   | Send page data to A.I.           | Send country view data to A.I.   |                                                                                                                                                                                                                                                                     |
| Send country view data to A.I.   | HTTP Request              | Send country views data to AI    | Send page Search data to A.I.    | Save A.I. output to Baserow      |                                                                                                                                                                                                                                                                     |
| Save A.I. output to Baserow      | Baserow                   | Save AI SEO reports into Baserow | Send country view data to A.I.   | None                             | ## Send data to Baserow Create a table first with the following columns: - Name - Country Views - Page Views - Search Report - Blog Enter the name of your website under "Blog" field.                                                                               |
| Sticky Note2                    | Sticky Note               | Instructions for AI credentials  | None                             | None                             | ## Send data to A.I. Fill in your Openrouter A.I. credentials. Use Header Auth. - Username: Authorization - Password: Bearer {insert your API key} Remember to add a space after bearer. Also, feel free to modify the prompt to A.1.                                   |
| Sticky Note3                    | Sticky Note               | Instructions for Baserow setup   | None                             | None                             | ## Send data to Baserow Create a table first with the following columns: - Name - Country Views - Page Views - Search Report - Blog Enter the name of your website under "Blog" field.                                                                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Nodes:**
   - Add a **Schedule Trigger** node.
     - Set interval to "1 week" to run weekly.
   - Add a **Manual Trigger** node for testing.
   
2. **Add Google Analytics OAuth2 Credentials:**
   - In n8n credentials, create Google Analytics OAuth2 credential.
   - Use Google Cloud Console to create OAuth client credentials.
   - Link this credential to all Google Analytics nodes.

3. **Create Google Analytics Nodes for Page Engagement:**
   - Add **Google Analytics** node named "Get Page Engagement Stats for this week".
     - Set propertyId to your GA property ID.
     - Metrics: screenPageViews, activeUsers, screenPageViewsPerUser, eventCount.
     - Dimensions: unifiedScreenName.
     - Return all data enabled.
     - Connect trigger nodes to this node.
   - Add **Google Analytics** node named "Get Page Engagement Stats for prior week".
     - Use custom date range from 14 days ago (start) to 7 days ago (end).
     - Same metrics and dimensions as above.
     - Connect “Parse data from Google Analytics” node to this node.

4. **Add Code Node "Parse data from Google Analytics":**
   - Insert JavaScript code to validate and transform GA response rows into simplified objects.
   - Output URL-encoded JSON string as `urlString`.
   - Connect "Get Page Engagement Stats for this week" to this node.
   - Connect this node to "Get Page Engagement Stats for prior week".

5. **Add Code Node "Parse GA data":**
   - Similar JavaScript code for prior week page engagement data.
   - Connect "Get Page Engagement Stats for prior week" to this node.

6. **Create Google Analytics Nodes for Google Search Console Data:**
   - Add **Google Analytics** node "Get Google Search Results for this week".
     - Metrics: activeUsers, engagedSessions, engagementRate, eventCount, organicGoogleSearchAveragePosition, organicGoogleSearchClickThroughRate, organicGoogleSearchClicks, organicGoogleSearchImpressions.
     - Dimension: landingPagePlusQueryString.
     - Connect "Parse GA data" to this node.
   - Add **Google Analytics** node "Get Google Search Results for last week".
     - Same metrics and dimension.
     - Custom date range: 14 days ago to 7 days ago.
     - Connect "Parse Google Analytics Data" to this node.

7. **Add Code Nodes "Parse Google Analytics Data" and "Parse Google Analytics Data1":**
   - Similar validation and transformation JavaScript for search console data.
   - Connect the respective GA nodes to these code nodes.

8. **Create Google Analytics Nodes for Country Views:**
   - Add **Google Analytics** node "Get Country views data for this week".
     - Metrics: activeUsers, newUsers, engagementRate, engagedSessions, eventCount, sessions.
     - Dimension: country.
     - Connect "Parse Google Analytics Data1" to this node.
   - Add **Google Analytics** node "Get Country views data for last week".
     - Same metrics and dimension.
     - Custom date range: 14 days ago to 7 days ago.
     - Connect "Parse Google analytics data" to this node.

9. **Add Code Nodes "Parse Google analytics data" and "Parse Google analytics data1":**
   - JavaScript to parse country views data similarly.
   - Connect from respective GA nodes.

10. **Add HTTP Request Nodes to Send Data to AI:**
    - Add "Send page data to A.I." HTTP Request node.
      - POST to https://openrouter.ai/api/v1/chat/completions
      - Body JSON includes model and messages comparing last two weeks' page engagement data URL-encoded strings.
      - Authentication: HTTP Header Auth with header `Authorization: Bearer {API_KEY}`.
      - Connect from "Parse Google analytics data1".
    - Add "Send page Search data to A.I." node similarly.
      - Uses parsed Google Search console data.
      - Connect from "Send page data to A.I.".
    - Add "Send country view data to A.I." node similarly.
      - Uses parsed country views data.
      - Connect from "Send page Search data to A.I.".

11. **Add Baserow Node to Save AI Outputs:**
    - Add Baserow node "Save A.I. output to Baserow".
    - Configure with your Baserow API credentials.
    - Set database and table IDs properly.
    - Map fields: Name (string), Country Views, Page Views, Search Report (all text from AI), Blog (string).
    - Connect from "Send country view data to A.I.".

12. **Add Sticky Notes for Documentation:**
    - Add sticky notes describing:
      - Workflow purpose and instructions.
      - Google Analytics credential and property ID guidance.
      - AI credential usage (Openrouter API key with proper header).
      - Baserow table and field setup instructions.

13. **Test Workflow:**
    - Use manual trigger node to test execution.
    - Verify data retrieval, parsing, AI calls, and data storage.
    - Adjust credentials and property IDs as needed.

---

### 5. General Notes & Resources

| Note Content                                                                                                                       | Context or Link                                                                                                                      |
|-----------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------|
| Detailed case study on using AI to analyze Google Analytics data for SEO reporting with n8n automation.                          | https://rumjahn.com/how-i-used-a-i-to-be-an-seo-expert-and-analyzed-my-google-analytics-data-in-n8n-and-make-com/                   |
| Instructions for creating Google Analytics OAuth2 credentials for n8n integration.                                                | https://docs.n8n.io/integrations/builtin/credentials/google/oauth-single-service/                                                   |
| Guidance on finding and using Google Analytics property ID.                                                                       | https://developers.google.com/analytics/devguides/reporting/data/v1/property-id                                                     |
| Notes on configuring Openrouter.ai API authentication with HTTP Header Auth (Authorization: Bearer {API_KEY}).                    | Included in Sticky Note2                                                                                                            |
| Baserow setup requires creating a table with columns: Name, Country Views, Page Views, Search Report, Blog.                       | Included in Sticky Note3                                                                                                            |
| The prompt text sent to AI can be customized in HTTP Request nodes to tailor SEO suggestions or output format.                   | See HTTP Request node JSON body parameters                                                                                          |

---

This structured document enables developers and AI agents to fully understand, reproduce, and troubleshoot the given workflow, providing clear insight into each node’s purpose, configuration, and interconnections.