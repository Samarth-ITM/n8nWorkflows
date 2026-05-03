Conference Networking Intelligence from Eventbrite with ScrapeGraphAI

https://n8nworkflows.xyz/workflows/conference-networking-intelligence-from-eventbrite-with-scrapegraphai-6568


# Conference Networking Intelligence from Eventbrite with ScrapeGraphAI

### 1. Workflow Overview

This workflow, titled **Conference Networking Intelligence from Eventbrite with ScrapeGraphAI**, automates the process of discovering, analyzing, and generating actionable networking intelligence for industry conferences. Its primary use case is for professionals, event organizers, or business development teams seeking to optimize their conference attendance strategy by identifying key events, speakers, sessions, and networking opportunities.

The workflow is logically divided into five main blocks:

- **1.1 Weekly Trigger:** Automatically initiates the workflow on a weekly schedule to ensure up-to-date conference discovery.
- **1.2 Conference Discovery:** Scrapes Eventbrite to extract detailed conference information.
- **1.3 Speaker Intelligence:** Analyzes conference speakers to identify high-value contacts and relevant session details.
- **1.4 Agenda Analysis:** Extracts and processes the conference schedule, including sessions and networking breaks.
- **1.5 Networking Strategy AI:** Uses custom JavaScript logic to analyze all collected data and produce actionable networking recommendations.

---

### 2. Block-by-Block Analysis

#### 2.1 Weekly Trigger

- **Overview:**  
  This block sets up an automated weekly trigger to start the entire workflow, ensuring conference data is collected regularly for fresh insights.

- **Nodes Involved:**  
  - Schedule Trigger

- **Node Details:**

  - **Schedule Trigger**  
    - Type: Trigger node  
    - Role: Initiates the workflow on a fixed schedule.  
    - Configuration:  
      - Frequency: Every week (default Monday, 9 AM local timezone).  
      - Timezone: Uses local timezone automatically.  
    - Input Connections: None (trigger node).  
    - Output Connections: Connected to Conference Scraper node.  
    - Edge Cases:  
      - If n8n server time is misconfigured, schedule may trigger at unexpected times.  
      - Network or server downtime could delay triggering.  
    - Notes:  
      - Allows adjustment to daily, bi-weekly, or monthly frequency based on user needs.

#### 2.2 Conference Discovery

- **Overview:**  
  Scrapes Eventbrite’s business conferences listing page for San Francisco, CA, extracting detailed structured data about upcoming conferences.

- **Nodes Involved:**  
  - Conference Scraper

- **Node Details:**

  - **Conference Scraper**  
    - Type: ScrapeGraphAI node (web scraping with AI-powered extraction)  
    - Role: Extracts conference metadata from the Eventbrite page.  
    - Configuration:  
      - User prompt defines a JSON schema specifying fields such as conference name, date, location, venue, description, ticket price, organizer, URLs, categories, and estimated attendance.  
      - Target URL: Eventbrite business conferences in San Francisco (static URL).  
    - Input Connections: Receives trigger from Schedule Trigger.  
    - Output Connections: Feeds extracted data into Speaker Analyzer.  
    - Key Expressions: None dynamic; fixed URL and prompt.  
    - Edge Cases:  
      - Changes to Eventbrite page structure may cause scraping failure or incorrect extraction.  
      - Rate limits or CAPTCHA on Eventbrite could block scraping.  
      - If no conferences found, downstream nodes may receive empty data.  
    - Notes:  
      - Supports extension to other event listing sources by adjusting URL and prompt.

#### 2.3 Speaker Intelligence

- **Overview:**  
  Scrapes the conference’s website URL (dynamically obtained) to extract detailed speaker information with professional profiles, session data, and prioritization.

- **Nodes Involved:**  
  - Speaker Analyzer

- **Node Details:**

  - **Speaker Analyzer**  
    - Type: ScrapeGraphAI node  
    - Role: Extracts speaker details from the conference website.  
    - Configuration:  
      - User prompt defines JSON schema for speakers with fields like name, title, company, bio, LinkedIn URL, session title/time/track, expertise areas, and contact priority.  
      - Target URL: Dynamically set via expression `={{ $json.website_url }}` from Conference Scraper output.  
    - Input Connections: Input from Conference Scraper node.  
    - Output Connections: Connects to Agenda Parser.  
    - Key Expressions: URL expression referencing previous node’s output.  
    - Edge Cases:  
      - If the conference website URL is missing or malformed, scraping fails.  
      - Website content changes or anti-scraping defenses may affect accuracy.  
      - Empty or incomplete speaker data leads to partial results.  
    - Version Requirements: Uses dynamic expression support (n8n version 0.152+ recommended).  

#### 2.4 Agenda Analysis

- **Overview:**  
  Extracts the full conference agenda and networking opportunities from the conference website, mapping session details and breaks for strategic planning.

- **Nodes Involved:**  
  - Agenda Parser

- **Node Details:**

  - **Agenda Parser**  
    - Type: ScrapeGraphAI node  
    - Role: Extracts agenda including sessions, speakers, tracks, session types, topics, and networking breaks.  
    - Configuration:  
      - User prompt specifies JSON schema containing agenda items and networking opportunities.  
      - Target URL uses expression `={{ $('Conference Scraper').item.json.website_url }}` referencing conference URL.  
    - Input Connections: Receives input from Speaker Analyzer.  
    - Output Connections: Feeds into Networking Opportunity Finder (code node).  
    - Key Expressions: Uses expression to dynamically obtain URL from earlier node.  
    - Edge Cases:  
      - Conference website changes or missing agenda pages may cause incomplete data.  
      - Complex agenda structures (multiple tracks, overlapping sessions) may require prompt tuning.  
      - Network or scraping failures could return no data.  

#### 2.5 Networking Strategy AI

- **Overview:**  
  Processes all prior extracted data to identify key networking opportunities, prioritize contacts, and recommend strategic session attendance with rationale.

- **Nodes Involved:**  
  - Networking Opportunity Finder (Code node)

- **Node Details:**

  - **Networking Opportunity Finder**  
    - Type: Code node (JavaScript)  
    - Role: Combines conference data, speaker intelligence, and agenda to generate structured networking recommendations.  
    - Configuration: Custom JavaScript code that:  
      - Filters high-priority speakers (contact_priority = High).  
      - Identifies networking breaks and coffee sessions from agenda.  
      - Detects strategic sessions such as Panels, Workshops, or sessions with topics like AI, Innovation, Strategy, Leadership.  
      - Constructs detailed opportunity objects including approach advice and LinkedIn URLs.  
      - Returns an analysis object summarizing conference overview, networking opportunities, strategic sessions, key speakers, and total opportunities count.  
    - Input Connections: Receives data from Agenda Parser (which depends on Speaker Analyzer and Conference Scraper).  
    - Output Connections: None (end node).  
    - Edge Cases:  
      - Missing or malformed input JSON may cause runtime errors.  
      - Empty or incomplete data sets reduce usefulness of output.  
      - Logic assumes certain schema presence; schema drift breaks code.  
    - Notes:  
      - Console logs total opportunities count for monitoring.  
      - Designed for extensibility to add further networking heuristics.

---

### 3. Summary Table

| Node Name                  | Node Type                 | Functional Role                   | Input Node(s)          | Output Node(s)           | Sticky Note                                                                                                    |
|----------------------------|---------------------------|---------------------------------|-----------------------|--------------------------|----------------------------------------------------------------------------------------------------------------|
| Schedule Trigger           | scheduleTrigger (Trigger) | Initiates workflow weekly       | None                  | Conference Scraper       | Step 1: Weekly Trigger ⏰ - Automates weekly conference search with customizable frequency.                    |
| Conference Scraper         | scrapegraphAi             | Extracts conference data        | Schedule Trigger      | Speaker Analyzer         | Step 2: Conference Discovery 🔍 - Extracts detailed conference metadata from Eventbrite page.                  |
| Speaker Analyzer           | scrapegraphAi             | Extracts speaker intelligence   | Conference Scraper    | Agenda Parser            | Step 3: Speaker Intelligence 🎤 - Analyzes speakers for networking prioritization and profile data.            |
| Agenda Parser              | scrapegraphAi             | Extracts conference agenda      | Speaker Analyzer      | Networking Opportunity Finder | Step 4: Agenda Analysis 📅 - Maps sessions, tracks, and networking breaks for strategic planning.              |
| Networking Opportunity Finder | code                     | Generates networking strategy   | Agenda Parser         | None                     | Step 5: Networking Strategy AI 🤝 - AI-powered analysis to maximize networking ROI and recommend approaches.    |
| Sticky Note 1              | stickyNote                | Documentation note              | None                  | None                     | Step 1: Weekly Trigger ⏰ - Explains weekly trigger rationale and options.                                      |
| Sticky Note 2              | stickyNote                | Documentation note              | None                  | None                     | Step 2: Conference Discovery 🔍 - Details data extracted and supported sources.                                |
| Sticky Note 3              | stickyNote                | Documentation note              | None                  | None                     | Step 3: Speaker Intelligence 🎤 - Explains speaker analysis and prioritization criteria.                       |
| Sticky Note 4              | stickyNote                | Documentation note              | None                  | None                     | Step 4: Agenda Analysis 📅 - Details agenda extraction and session mapping.                                    |
| Sticky Note 5              | stickyNote                | Documentation note              | None                  | None                     | Step 5: Networking Strategy AI 🤝 - Describes AI analysis goals and output intelligence.                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a Schedule Trigger node:**  
   - Type: Schedule Trigger  
   - Parameters:  
     - Set interval to "Weekly".  
     - Default: Every Monday at 9:00 AM, local timezone.  
   - No credentials needed.  
   - Position: Leftmost node (e.g., [-1700, 900]).

3. **Add a ScrapeGraphAI node named "Conference Scraper":**  
   - Type: scrapegraphAi  
   - Parameters:  
     - User Prompt:  
       ```
       Extract conference information from this page. Use the following schema for response: {
         "conference_name": "Tech Summit 2024",
         "date": "2024-08-15",
         "location": "San Francisco, CA",
         "venue": "Moscone Center",
         "description": "Annual technology conference",
         "ticket_price": "$299",
         "organizer": "Tech Events Inc",
         "website_url": "https://example.com/conference",
         "registration_url": "https://example.com/register",
         "categories": ["Technology", "Business"],
         "estimated_attendees": "500-1000"
       }
       ```  
     - Website URL: `https://www.eventbrite.com/d/ca--san-francisco/business-conferences/` (static)  
   - Connect Schedule Trigger’s output to this node input.

4. **Add a ScrapeGraphAI node named "Speaker Analyzer":**  
   - Type: scrapegraphAi  
   - Parameters:  
     - User Prompt:  
       ```
       Extract speaker information from this conference website. Use the following schema: {
         "speakers": [{
           "name": "John Doe",
           "title": "CEO",
           "company": "Tech Corp",
           "bio": "Tech industry veteran",
           "linkedin_url": "https://linkedin.com/in/johndoe",
           "session_title": "Future of AI",
           "session_time": "10:00 AM",
           "session_track": "Main Stage",
           "expertise_areas": ["AI", "Machine Learning"],
           "contact_priority": "High"
         }]
       }
       ```  
     - Website URL: Use expression referencing `Conference Scraper` output: `={{ $json.website_url }}`  
   - Connect output of Conference Scraper to this node input.

5. **Add a ScrapeGraphAI node named "Agenda Parser":**  
   - Type: scrapegraphAi  
   - Parameters:  
     - User Prompt:  
       ```
       Extract the conference agenda and schedule. Use this schema: {
         "agenda": [{
           "time": "09:00 AM",
           "session_title": "Opening Keynote",
           "speaker": "Jane Smith",
           "track": "Main Stage",
           "duration": "60 minutes",
           "session_type": "Keynote",
           "topics": ["Industry Trends"],
           "networking_break": false,
           "location": "Hall A"
         }],
         "networking_opportunities": [{
           "type": "Coffee Break",
           "time": "10:30 AM",
           "duration": "30 minutes",
           "location": "Lobby"
         }]
       }
       ```  
     - Website URL: Use expression referencing `Conference Scraper` output: `={{ $('Conference Scraper').item.json.website_url }}`  
   - Connect Speaker Analyzer output to this node input.

6. **Add a Code node named "Networking Opportunity Finder":**  
   - Type: code  
   - Parameters: Insert the following JavaScript code:
     ```javascript
     // Get all input data
     const conferenceData = $('Conference Scraper').item.json;
     const speakerData = $('Speaker Analyzer').item.json;
     const agendaData = $('Agenda Parser').item.json;

     function analyzeNetworkingOpportunities(conference, speakers, agenda) {
       const opportunities = [];

       // Extract high-priority speakers
       const highPrioritySpeakers = speakers.speakers?.filter(s => s.contact_priority === 'High') || [];

       // Find networking breaks and coffee sessions
       const networkingTimes = agenda.networking_opportunities || [];

       // Identify strategic sessions to attend
       const strategicSessions = agenda.agenda?.filter(session =>
         session.session_type === 'Panel' ||
         session.session_type === 'Workshop' ||
         session.topics?.some(topic => ['AI', 'Innovation', 'Strategy', 'Leadership'].includes(topic))
       ) || [];

       // Generate networking recommendations
       highPrioritySpeakers.forEach(speaker => {
         opportunities.push({
           type: 'Speaker Meeting',
           target: speaker.name,
           company: speaker.company,
           session: speaker.session_title,
           time: speaker.session_time,
           priority: 'High',
           reason: `Industry expert in ${speaker.expertise_areas?.join(', ')}`,
           approach: `Attend their session: "${speaker.session_title}" and approach during Q&A or after`,
           linkedin: speaker.linkedin_url
         });
       });

       // Add general networking opportunities
       networkingTimes.forEach(netTime => {
         opportunities.push({
           type: 'General Networking',
           time: netTime.time,
           duration: netTime.duration,
           location: netTime.location,
           priority: 'Medium',
           approach: 'Casual conversations with attendees'
         });
       });

       return {
         conference_overview: {
           name: conference.conference_name,
           date: conference.date,
           location: conference.location,
           estimated_attendees: conference.estimated_attendees
         },
         networking_opportunities: opportunities,
         strategic_sessions: strategicSessions,
         key_speakers: highPrioritySpeakers,
         total_opportunities: opportunities.length
       };
     }

     const analysis = analyzeNetworkingOpportunities(conferenceData, speakerData, agendaData);

     console.log(`Found ${analysis.total_opportunities} networking opportunities`);

     return [{ json: analysis }];
     ```
   - Connect Agenda Parser output to this node input.

7. **Add Sticky Notes for documentation (optional):**  
   - Add five sticky note nodes, placing them nearby their related functional blocks with the exact content as described in the sticky notes in the original workflow for clarity.

8. **Save and Activate the workflow.**

---

### 5. General Notes & Resources

| Note Content                                                                                                 | Context or Link                                                                                   |
|--------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Conference announcements happen regularly; weekly scheduling balances freshness and resource use.             | Sticky Note 1: Explains scheduling rationale.                                                   |
| Eventbrite is a primary supported source but can be extended to Meetup, industry sites, or aggregator sites. | Sticky Note 2: Details supported conference sources.                                            |
| Speaker prioritization focuses on C-levels and industry thought leaders to maximize networking impact.        | Sticky Note 3: Explains speaker intelligence and prioritization.                                |
| Agenda extraction highlights sessions, tracks, and networking breaks to optimize planning.                   | Sticky Note 4: Describes agenda parsing and session mapping.                                    |
| AI-driven networking strategy includes contact ranking, approach recommendations, and timing optimization.   | Sticky Note 5: Details AI-powered networking analysis and output intelligence.                  |

---

This workflow is designed to be robust but relies on stable conference website structures and accessible URLs. Regular monitoring and prompt updates to scraping prompts may be necessary to adapt to website changes. Authentication is not required for the scraping nodes as they target publicly accessible pages. The JavaScript code node assumes data integrity and presence of expected JSON fields; consider adding validation or error handling for production use.

---

*Disclaimer: The provided text is exclusively derived from an automated workflow created with n8n, respecting all content policies. It contains no illegal, offensive, or protected elements. All manipulated data is legal and public.*