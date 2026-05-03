Automated Workflow for Daily LinkedIn Posts Using Notion

https://n8nworkflows.xyz/workflows/automated-workflow-for-daily-linkedin-posts-using-notion-2273


# Automated Workflow for Daily LinkedIn Posts Using Notion

### 1. Workflow Overview

This workflow automates daily LinkedIn posts by leveraging content stored in a Notion database. Each day, it fetches the scheduled post for that day from Notion, processes and formats the content—including downloading any associated images—and then publishes the post on LinkedIn. Finally, it updates the status of the post in the Notion database to mark it as published.

The workflow can be logically divided into these blocks:

- **1.1 Scheduled Trigger**: Initiates the workflow daily at a specified hour.
- **1.2 Fetching Post from Notion**: Queries the Notion database to retrieve the post scheduled for the current day and fetches its detailed content blocks.
- **1.3 Content Processing and Formatting**: Aggregates, formats the textual content, and downloads related images.
- **1.4 LinkedIn Publishing**: Posts the formatted content with images on LinkedIn.
- **1.5 Post Status Update**: Updates the original Notion post status to "Published" after successful posting.

Supporting sticky notes provide setup instructions and guidance at various stages.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger

**Overview:**  
This block triggers the entire workflow automatically each day at a specified time (3 PM).

**Nodes Involved:**  
- Schedule Trigger  
- Sticky Note (Start the flow every day)

**Node Details:**

- **Schedule Trigger**  
  - *Type & Role*: Built-in scheduler node that triggers workflow on a time basis.  
  - *Configuration*: Set to trigger daily at 15:00 (3 PM).  
  - *Expressions*: None.  
  - *Input/Output*: No input; outputs a trigger to the next node.  
  - *Failure modes*: Potential failure if n8n instance is offline at trigger time or timezone misconfiguration.  
  - *Version*: 1.2.

- **Sticky Note (Start the flow every day at the same time)**  
  - *Type*: Informational note for users.  
  - *Content*: "Start the flow every day at the same time."  
  - *Role*: Documentation only.

---

#### 1.2 Fetching Post from Notion

**Overview:**  
This block queries the Notion database to find the post scheduled for the current day and retrieves the content blocks to be used in the LinkedIn post.

**Nodes Involved:**  
- Filter the table for the day's post (Notion)  
- Fetch the content on the page (Notion)  
- Sticky Note1 (Fetch the day's post from my Notion database)

**Node Details:**

- **Filter the table for the day's post**  
  - *Type & Role*: Notion node to query database pages.  
  - *Configuration*: Filters pages where the "Date" property equals today's date using the expression `={{ $today.format("yyyy/mM/dd") }}`.  
  - *Input/Output*: Input from Schedule Trigger; outputs list of pages matching the date filter.  
  - *Credentials*: Uses configured Notion API credentials ("Notion Weck").  
  - *Failure modes*: No posts found for the day (empty output), API rate limits, or auth errors.  
  - *Version*: 2.2.

- **Fetch the content on the page**  
  - *Type & Role*: Notion node to retrieve all content blocks from a given page.  
  - *Configuration*: Uses the page URL from the previous node to fetch all blocks. Returns all blocks.  
  - *Input/Output*: Input from previous node; outputs page content blocks.  
  - *Credentials*: Same Notion credentials as above.  
  - *Failure modes*: Invalid URL, API limits, or content retrieval errors.  
  - *Version*: 2.2.

- **Sticky Note1**  
  - *Content*: "Fetch the day's post from my Notion database."  
  - *Role*: Documentation.

---

#### 1.3 Content Processing and Formatting

**Overview:**  
Aggregates the content blocks from Notion, formats the text for LinkedIn post style, downloads the first image if present, and prepares merged data for posting.

**Nodes Involved:**  
- Aggregate the Notion blocks (Aggregate)  
- Format the post (Code)  
- Download image (HTTP Request)  
- Merge  
- Sticky Note2 (Process and format the post)

**Node Details:**

- **Aggregate the Notion blocks**  
  - *Type & Role*: Aggregates multiple fields from input items into arrays.  
  - *Configuration*: Aggregates `content` and `image.file.url` fields from the Notion blocks.  
  - *Input/Output*: Input from "Fetch the content on the page"; outputs aggregated arrays of content and image URLs.  
  - *Failure modes*: No content or image fields present in blocks.  
  - *Version*: 1.

- **Format the post**  
  - *Type & Role*: Custom JavaScript code node to concatenate and format the post text.  
  - *Configuration*:  
    - Takes the first Notion content array from input.  
    - Concatenates lines into a formatted string, adding line breaks and extra breaks before list items (lines starting with '-').  
    - Outputs object with formattedText property.  
  - *Key expression snippet*:  
    ```js
    const notionData = items[0].json.content;

    let formattedText = notionData[0];

    for (let i = 1; i < notionData.length; i++) {
        if (notionData[i].startsWith('-')) {
            formattedText += '\n\n' + notionData[i];
        } else {
            formattedText += '\n' + notionData[i];
        }
    }

    return [{ formattedText: formattedText }];
    ```  
  - *Input/Output*: Input from aggregate node; output is formatted text for LinkedIn.  
  - *Failure modes*: Empty or malformed content array, code errors.  
  - *Version*: 2.

- **Download image**  
  - *Type & Role*: HTTP Request node to download image by URL.  
  - *Configuration*: Uses the first image URL from aggregated data (`={{ $json.url[0] }}`).  
  - *Input/Output*: Input from aggregate node; output is binary image data.  
  - *Failure modes*: No image URL available, HTTP errors, timeouts.  
  - *Version*: 4.2.

- **Merge**  
  - *Type & Role*: Combines outputs from "Format the post" and "Download image" nodes by position to create a single data item for LinkedIn.  
  - *Configuration*: Mode set to "combine" with "mergeByPosition" to pair corresponding outputs.  
  - *Input/Output*: Inputs from "Format the post" (index 0) and "Download image" (index 1); output to LinkedIn node.  
  - *Failure modes*: Mismatched array lengths, missing inputs.  
  - *Version*: 2.1.

- **Sticky Note2**  
  - *Content*: "Process and format the post."  
  - *Role*: Documentation.

---

#### 1.4 LinkedIn Publishing

**Overview:**  
Posts the formatted text and image on LinkedIn using the LinkedIn node, targeting a specific personal or company profile.

**Nodes Involved:**  
- Publish on LinkedIn  
- Sticky Note3 (Setup instructions about LinkedIn credentials)

**Node Details:**

- **Publish on LinkedIn**  
  - *Type & Role*: LinkedIn node to publish a post.  
  - *Configuration*:  
    - Text field set to the formatted text from the previous merge node (`={{ $json.formattedText }}`).  
    - Person ID hardcoded as "CcS-_lLyzG" (must correspond to a valid LinkedIn profile or company page).  
    - Share media category set to "IMAGE" to include the downloaded image.  
  - *Credentials*: Uses LinkedIn OAuth2 credentials ("LinkedIn account").  
  - *Input/Output*: Input from Merge node; output to update Notion node.  
  - *Failure modes*: Auth errors, invalid person ID, API limits, image upload failures.  
  - *Version*: 1.

- **Sticky Note3**  
  - *Content*:  
    ```
    1. Setup
    Set up your Notion and LinkedIn credentials.
    Attention to the LinkedIn credential: to post on your personal or company profile, you need to have a company page assigned to your profile. After that, you can choose where you want to post.
    ```  
  - *Role*: Setup instructions especially about LinkedIn credential requirements.

---

#### 1.5 Post Status Update

**Overview:**  
After successful publishing, this block updates the Notion database entry for the post to mark its status as "Published."

**Nodes Involved:**  
- Update post status in notion database

**Node Details:**

- **Update post status in notion database**  
  - *Type & Role*: Notion node to update a database page property.  
  - *Configuration*:  
    - Uses page ID of the post from "Filter the table for the day's post" node (`={{ $('Filter the table for the day\'s post').item.json.url }}`).  
    - Updates the "Status" property to select value "Published."  
  - *Credentials*: Same Notion credentials as above.  
  - *Input/Output*: Input from LinkedIn publish node; output is the update confirmation.  
  - *Failure modes*: API errors, invalid page ID, permission errors.  
  - *Version*: 2.2.

---

### 3. Summary Table

| Node Name                            | Node Type              | Functional Role                          | Input Node(s)                      | Output Node(s)                        | Sticky Note                                                  |
|------------------------------------|------------------------|----------------------------------------|----------------------------------|-------------------------------------|--------------------------------------------------------------|
| Schedule Trigger                   | Schedule Trigger       | Initiate workflow daily at 3 PM          | -                                | Filter the table for the day's post | Start the flow every day at the same time                    |
| Filter the table for the day's post| Notion                 | Query Notion database for today's post   | Schedule Trigger                 | Fetch the content on the page        | Fetch the day's post from my Notion database                 |
| Fetch the content on the page       | Notion                 | Retrieve all content blocks of post      | Filter the table for the day's post | Aggregate the Notion blocks         | Fetch the day's post from my Notion database                 |
| Aggregate the Notion blocks         | Aggregate              | Aggregate content and image URLs         | Fetch the content on the page    | Format the post, Download image      | Process and format the post                                   |
| Format the post                    | Code                   | Format Notion content for LinkedIn post  | Aggregate the Notion blocks      | Merge                               | Process and format the post                                   |
| Download image                    | HTTP Request           | Download first image from Notion post    | Aggregate the Notion blocks      | Merge                               | Process and format the post                                   |
| Merge                             | Merge                  | Combine formatted text and image for post| Format the post, Download image  | Publish on LinkedIn                 | Process and format the post                                   |
| Publish on LinkedIn                | LinkedIn                | Post content and image on LinkedIn       | Merge                           | Update post status in notion database | Setup instructions about LinkedIn credentials                |
| Update post status in notion database | Notion                 | Mark post as "Published" in Notion DB    | Publish on LinkedIn             | -                                   |                                                              |
| Sticky Note                       | Sticky Note            | Documentation                            | -                                | -                                   | Start the flow every day at the same time                    |
| Sticky Note1                      | Sticky Note            | Documentation                            | -                                | -                                   | Fetch the day's post from my Notion database                 |
| Sticky Note2                      | Sticky Note            | Documentation                            | -                                | -                                   | Process and format the post                                   |
| Sticky Note3                      | Sticky Note            | Documentation                            | -                                | -                                   | 1. Setup LinkedIn and Notion credentials; company page note |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node**  
   - Type: Schedule Trigger  
   - Set to trigger daily at 15:00 (3 PM).

2. **Create Notion Node: Filter the table for the day's post**  
   - Type: Notion (Database Page Get All)  
   - Credentials: Connect your Notion API credentials.  
   - Operation: Get all pages from your target database (use your Notion database ID).  
   - Filter: Add a condition where the "Date" property equals today's date using expression: `={{ $today.format("yyyy/mM/dd") }}`.  
   - Connect Schedule Trigger output to this node.

3. **Create Notion Node: Fetch the content on the page**  
   - Type: Notion (Block Get All)  
   - Credentials: Same Notion API credentials.  
   - Operation: Get all blocks from the page.  
   - Set `blockId` parameter using the URL from the previous node with expression: `={{ $json.url }}`.  
   - Connect output of "Filter the table for the day's post" to this node.

4. **Create Aggregate Node: Aggregate the Notion blocks**  
   - Type: Aggregate  
   - Configure to aggregate fields:  
     - `content` field from blocks  
     - `image.file.url` field from blocks  
   - Connect output of "Fetch the content on the page" to this node.

5. **Create Code Node: Format the post**  
   - Type: Code  
   - Language: JavaScript  
   - Paste the following code:
     ```js
     const notionData = items[0].json.content;

     let formattedText = notionData[0];

     for (let i = 1; i < notionData.length; i++) {
         if (notionData[i].startsWith('-')) {
             formattedText += '\n\n' + notionData[i];
         } else {
             formattedText += '\n' + notionData[i];
         }
     }

     return [{ formattedText: formattedText }];
     ```
   - Connect output of Aggregate node to this node.

6. **Create HTTP Request Node: Download image**  
   - Type: HTTP Request  
   - URL parameter set to first image URL with expression: `={{ $json.url[0] }}`.  
   - Connect output of Aggregate node to this node.

7. **Create Merge Node**  
   - Type: Merge  
   - Mode: Combine  
   - Combination Mode: Merge by Position  
   - Connect output of "Format the post" node to first input (index 0).  
   - Connect output of "Download image" node to second input (index 1).

8. **Create LinkedIn Node: Publish on LinkedIn**  
   - Type: LinkedIn  
   - Credentials: OAuth2 credentials for LinkedIn account.  
   - Text: Set expression to `={{ $json.formattedText }}` from Merge node.  
   - Person: Enter your LinkedIn person or company ID (e.g., "CcS-_lLyzG").  
   - Share Media Category: Set to "IMAGE" to post with image.  
   - Connect output of Merge node to this node.

9. **Create Notion Node: Update post status in notion database**  
   - Type: Notion (Database Page Update)  
   - Credentials: Same Notion API credentials.  
   - Page ID: Use expression to get page URL from "Filter the table for the day's post" node: `={{ $('Filter the table for the day\'s post').item.json.url }}`  
   - Properties to update: Set "Status" select property to "Published".  
   - Connect output of LinkedIn node to this node.

10. **Add Sticky Notes (Optional but recommended for clarity)**  
    - Add notes near respective blocks describing their roles and setup instructions as in the original workflow.

11. **Set Workflow Settings**  
    - Execution timeout: 30 seconds (adjust if needed).  
    - Save manual executions if necessary.

12. **Test workflow**  
    - Manually trigger or wait for scheduled time to verify correct operation.  
    - Handle any authentication or API errors as they arise.

---

### 5. General Notes & Resources

| Note Content                                                                                              | Context or Link                              |
|----------------------------------------------------------------------------------------------------------|----------------------------------------------|
| To post on LinkedIn personal or company profiles, ensure your LinkedIn OAuth2 credential is linked to a company page or profile you have publishing rights for. | Sticky Note3 in workflow                      |
| The Notion database should have a "Date" property of type date and a "Status" select property with value "Published" to track posting status. | Workflow configuration requirement            |
| For detailed Notion API usage, refer to official docs: https://developers.notion.com/reference/intro      | External API documentation                      |
| LinkedIn API documentation for posting content: https://learn.microsoft.com/en-us/linkedin/marketing/integrations/community-management/shares/share-api | External API documentation                      |

---

This documentation provides a complete understanding of the workflow's logic, configuration, potential failure points, and stepwise instructions for reproduction or modification.