College Football Analytics Platform with Comprehensive Data API Access

https://n8nworkflows.xyz/workflows/college-football-analytics-platform-with-comprehensive-data-api-access-5493


# College Football Analytics Platform with Comprehensive Data API Access

### 1. Workflow Overview

This workflow, titled **College Football Data API MCP Server**, provides a comprehensive API interface for accessing detailed college football data. It is designed as a server endpoint compatible with AI agents using the MCP (Model-Controller-Provider) framework, enabling them to invoke any of 51 different operations related to college football analytics.

**Target Use Cases:**  
- AI agents or applications requiring structured access to a wide range of college football data  
- Developers wanting to integrate college football statistics, player info, game details, rankings, and metrics into their systems  
- Data analysts and sports researchers looking for automated API access to CollegeFootballData.com  

**Logical Blocks:**  
The workflow is modularly organized by data domain, with each block representing a set of API endpoints focused on a thematic area:  

- 1.1 MCP Input Trigger  
- 1.2 Games Data  
- 1.3 Coaches Data  
- 1.4 Conferences Data  
- 1.5 Draft Data  
- 1.6 Drives Data  
- 1.7 Betting Data  
- 1.8 Plays Data  
- 1.9 Metrics Data  
- 1.10 Players Data  
- 1.11 Rankings Data  
- 1.12 Ratings Data  
- 1.13 Recruiting Data  
- 1.14 Teams Data  
- 1.15 Stats Data  
- 1.16 Venues Data  

Each block contains one or more HTTP Request Tool nodes that call specific CollegeFootballData.com API endpoints, with parameters automatically populated via `$fromAI()` expressions to allow dynamic queries based on AI agent input.

---

### 2. Block-by-Block Analysis

---

#### 1.1 MCP Input Trigger

- **Overview:**  
  Entry point for all AI agent requests. Acts as a webhook endpoint that receives API calls and routes them to the appropriate HTTP request nodes.

- **Nodes Involved:**  
  - College Football Data MCP Server

- **Node Details:**  
  - **Type:** MCP Trigger (from n8n-nodes-langchain)  
  - **Configuration:**  
    - Path: `college-football-data-mcp`  
    - Authentication: Header authentication with API key (`Authorization` header, expects `Bearer your_key`)  
  - **Input/Output:**  
    - Input: External HTTP requests from AI agents  
    - Output: Dispatches to all HTTP Request Tool nodes via ai_tool connections  
  - **Failure Modes:**  
    - Authentication failures if API key missing or invalid  
    - Webhook timeout or network issues  
  - **Notes:** This node orchestrates the entire workflow by exposing the API interface.

---

#### 1.2 Games Data

- **Overview:**  
  Provides access to various game-related endpoints including season calendars, advanced box scores, game results, media info, player stats, team stats, weather, records, and live scores.

- **Nodes Involved:**  
  - Season calendar  
  - Advanced box scores  
  - Games and results  
  - Game media information and schedules  
  - Player game stats  
  - Team game stats  
  - Game weather information (Patreon only)  
  - Team records  
  - Live game results (Patreon only)  

- **Node Details Example (Games and results):**  
  - **Type:** HTTP Request Tool  
  - **Configuration:** Calls `https://api.collegefootballdata.com/games` with query parameters populated from AI inputs (year, week, seasonType, team, home, away, conference, division, id)  
  - **Parameters:** All optional except year, dynamically filled from AI context using `$fromAI()`  
  - **Failure Modes:**  
    - Missing required parameters  
    - API rate limits  
    - Network errors  
  - **Notes:** Similar structure applies for all other nodes in this block, each calling distinct endpoints with relevant parameters.

---

#### 1.3 Coaches Data

- **Overview:**  
  Access to coaching records and history filtered by name, team, or year.

- **Nodes Involved:**  
  - Coaching records and history  

- **Node Details:**  
  - Calls `https://api.collegefootballdata.com/coaches` with optional query parameters (firstName, lastName, team, year, minYear, maxYear) from AI inputs.  
  - Failure modes include invalid parameter values or API errors.

---

#### 1.4 Conferences Data

- **Overview:**  
  Retrieves conference information.

- **Nodes Involved:**  
  - Conferences  

- **Node Details:**  
  - Calls `https://api.collegefootballdata.com/conferences` without parameters.  
  - Simple fetch, minimal failure risk except network issues.

---

#### 1.5 Draft Data

- **Overview:**  
  NFL draft picks, positions, and teams data.

- **Nodes Involved:**  
  - List of NFL Draft picks  
  - List of NFL positions  
  - List of NFL teams  

- **Node Details:**  
  - Each node calls related endpoints with optional filters (year, nflTeam, college, conference, position).  
  - Parameters dynamically populated.  
  - Failure modes: invalid filters, API errors.

---

#### 1.6 Drives Data

- **Overview:**  
  Access drive data and results with filters on season, team, offense, defense, conference, and classification.

- **Nodes Involved:**  
  - Drive data and results  

- **Node Details:**  
  - Calls `https://api.collegefootballdata.com/drives` with query parameters from AI.  
  - Requires year parameter; others optional.  
  - Failure modes: missing year, API errors.

---

#### 1.7 Betting Data

- **Overview:**  
  Betting lines and related data for games.

- **Nodes Involved:**  
  - Betting lines  

- **Node Details:**  
  - Calls `https://api.collegefootballdata.com/lines` with filters such as gameId, year, week, seasonType, team, home, away, conference.  
  - Parameters from AI context.  
  - Failure modes include invalid gameId or missing parameters.

---

#### 1.8 Plays Data

- **Overview:**  
  Play-by-play data, play types, player play stats, live metrics (Patreon only).

- **Nodes Involved:**  
  - Live metrics and PBP (Patreon only)  
  - Types of player play stats  
  - Play stats by play  
  - Play types  
  - Play by play data  

- **Node Details:**  
  - Multiple endpoints covering detailed play stats and types.  
  - Parameters include id, year, week, team, athleteId, statTypeId, seasonType, conference, classification.  
  - Failure modes: missing required parameters, Patreon access restriction.

---

#### 1.9 Metrics Data

- **Overview:**  
  Data on win probabilities, predicted points, and predicted points added (PPA/EPA) for players and teams.

- **Nodes Involved:**  
  - Win probability chart data  
  - Pregame win probability data  
  - Team Predicated Points Added (PPA/EPA) by game  
  - Player Predicated Points Added broken down by game  
  - Player Predicated Points Added broken down by season  
  - Predicted Points (Expected Points or EP)  
  - Predicted Points Added (PPA/EPA) data by team  

- **Node Details:**  
  - Calls various metrics endpoints with parameters such as gameId, year, week, team, conference, position, playerId, threshold, excludeGarbageTime, down, distance.  
  - Parameters populate dynamically.  
  - Failure modes: required parameters missing, API errors.

---

#### 1.10 Players Data

- **Overview:**  
  Player info including transfer portal, returning production, search, usage metrics, and player stats by season.

- **Nodes Involved:**  
  - Transfer portal by season  
  - Team returning production metrics  
  - Search for player information  
  - Player usage metrics broken down by season  
  - Player stats by season  

- **Node Details:**  
  - Endpoints accept filters like year, team, conference, position, searchTerm, playerId, excludeGarbageTime, category.  
  - Failure modes: invalid search terms, missing required params.

---

#### 1.11 Rankings Data

- **Overview:**  
  Historical polls and rankings data.

- **Nodes Involved:**  
  - Historical polls and rankings  

- **Node Details:**  
  - Calls `https://api.collegefootballdata.com/rankings` with year, week, seasonType parameters.  
  - Failure modes: missing year parameter.

---

#### 1.12 Ratings Data

- **Overview:**  
  Team ratings including Elo, SP+, and SRS ratings by season and conference.

- **Nodes Involved:**  
  - Historical Elo ratings  
  - Historical SP+ ratings  
  - Historical SP+ ratings by conference  
  - Historical SRS ratings  

- **Node Details:**  
  - Parameters include year, week, team, conference.  
  - Calls respective endpoints with dynamic parameters.  
  - Failure modes: missing required filters.

---

#### 1.13 Recruiting Data

- **Overview:**  
  Recruiting group ratings, player recruiting ratings and rankings, and team recruiting rankings.

- **Nodes Involved:**  
  - Recruit position group ratings  
  - Player recruiting ratings and rankings  
  - Team recruiting rankings and ratings  

- **Node Details:**  
  - Accept parameters such as startYear, endYear, year, classification, position, state, team, conference.  
  - Failure modes: conflicting or missing parameters.

---

#### 1.14 Teams Data

- **Overview:**  
  Team rosters, talent rankings, team info, FBS team list, and matchup history.

- **Nodes Involved:**  
  - Team rosters  
  - Team talent composite rankings  
  - Team information  
  - FBS team list  
  - Team matchup history  

- **Node Details:**  
  - Parameters include team, year, conference, team1, team2, minYear, maxYear.  
  - Failure modes: missing required parameters (e.g., team names).

---

#### 1.15 Stats Data

- **Overview:**  
  Statistical data on teams, including categories, advanced metrics by game and season, and season statistics.

- **Nodes Involved:**  
  - Team stat categories  
  - Advanced team metrics by game  
  - Team statistics by season  
  - Advanced team metrics by season  

- **Node Details:**  
  - Parameters like year, week, team, opponent, excludeGarbageTime, startWeek, endWeek, seasonType.  
  - Failure modes: missing year or team filters as required.

---

#### 1.16 Venues Data

- **Overview:**  
  Information about football arenas and venues.

- **Nodes Involved:**  
  - Arena and venue information  

- **Node Details:**  
  - Simple endpoint call without parameters.  
  - Failure modes: network issues only.

---

### 3. Summary Table

| Node Name                                | Node Type               | Functional Role                          | Input Node(s)                  | Output Node(s)                | Sticky Note                                                                                                               |
|-----------------------------------------|-------------------------|----------------------------------------|-------------------------------|------------------------------|---------------------------------------------------------------------------------------------------------------------------|
| Workflow Overview                       | Sticky Note             | Workflow description and instructions  |                               |                              | ## 🛠️ College Football Data MCP Server ✅ 51 operations ... (full content as in Workflow Overview)                         |
| College Football Data MCP Server        | MCP Trigger             | API entry point for AI agent requests  |                               | All HTTP Request Tool nodes   |                                                                                                                           |
| Sticky Note                            | Sticky Note             | Label for Games block                   |                               |                              | ## Games                                                                                                                  |
| Season calendar                        | HTTP Request Tool       | Fetch season calendar                   | College Football Data MCP Server |                             |                                                                                                                           |
| Advanced box scores                    | HTTP Request Tool       | Fetch advanced box scores               | College Football Data MCP Server |                             |                                                                                                                           |
| Games and results                     | HTTP Request Tool       | Fetch games and results                 | College Football Data MCP Server |                             |                                                                                                                           |
| Game media information and schedules  | HTTP Request Tool       | Fetch game media info and schedules    | College Football Data MCP Server |                             |                                                                                                                           |
| Player game stats                     | HTTP Request Tool       | Fetch player stats per game             | College Football Data MCP Server |                             |                                                                                                                           |
| Team game stats                      | HTTP Request Tool       | Fetch team stats per game               | College Football Data MCP Server |                             |                                                                                                                           |
| Game weather information (Patreon only) | HTTP Request Tool       | Fetch game weather info (Patreon)      | College Football Data MCP Server |                             |                                                                                                                           |
| Team records                        | HTTP Request Tool       | Fetch team records                      | College Football Data MCP Server |                             |                                                                                                                           |
| Live game results (Patreon only)   | HTTP Request Tool       | Fetch live game scores (Patreon)       | College Football Data MCP Server |                             |                                                                                                                           |
| Description - games                 | Sticky Note             | Description label for Games block      |                               |                              | ## 📋 Games - Games scores and statistics                                                                                 |
| Sticky Note2                       | Sticky Note             | Label for Coaches block                 |                               |                              | ## Coaches                                                                                                                |
| Coaching records and history       | HTTP Request Tool       | Fetch coaching records and history     | College Football Data MCP Server |                             |                                                                                                                           |
| Description - coaches             | Sticky Note             | Description label for Coaches block    |                               |                              | ## 📋 Coaches - Information about coaches                                                                                 |
| Sticky Note3                      | Sticky Note             | Label for Conferences block             |                               |                              | ## Conferences                                                                                                            |
| Conferences                      | HTTP Request Tool       | Fetch conference info                   | College Football Data MCP Server |                             |                                                                                                                           |
| Description - conferences        | Sticky Note             | Description label for Conferences block |                               |                              | ## 📋 Conferences - Conference information                                                                                 |
| Sticky Note4                      | Sticky Note             | Label for Draft block                   |                               |                              | ## Draft                                                                                                                  |
| List of NFL Draft picks           | HTTP Request Tool       | Fetch NFL draft picks                   | College Football Data MCP Server |                             |                                                                                                                           |
| List of NFL positions            | HTTP Request Tool       | Fetch NFL positions list                | College Football Data MCP Server |                             |                                                                                                                           |
| List of NFL teams                | HTTP Request Tool       | Fetch NFL teams list                    | College Football Data MCP Server |                             |                                                                                                                           |
| Description - draft             | Sticky Note             | Description label for Draft block      |                               |                              | ## 📋 Draft - NFL Draft data                                                                                              |
| Sticky Note5                     | Sticky Note             | Label for Drives block                  |                               |                              | ## Drives                                                                                                                 |
| Drive data and results          | HTTP Request Tool       | Fetch drive data and results            | College Football Data MCP Server |                             |                                                                                                                           |
| Description - drives           | Sticky Note             | Description label for Drives block     |                               |                              | ## 📋 Drives - Drive data                                                                                                |
| Sticky Note6                     | Sticky Note             | Label for Betting block                 |                               |                              | ## Betting                                                                                                                |
| Betting lines                 | HTTP Request Tool       | Fetch betting lines                     | College Football Data MCP Server |                             |                                                                                                                           |
| Description - betting         | Sticky Note             | Description label for Betting block    |                               |                              | ## 📋 Betting - Betting lines and data                                                                                   |
| Sticky Note7                     | Sticky Note             | Label for Plays block                   |                               |                              | ## Plays                                                                                                                  |
| Live metrics and PBP (Patreon only) | HTTP Request Tool       | Fetch live play-by-play metrics (Patreon) | College Football Data MCP Server |                             |                                                                                                                           |
| Types of player play stats     | HTTP Request Tool       | Fetch types of player play stats       | College Football Data MCP Server |                             |                                                                                                                           |
| Play stats by play            | HTTP Request Tool       | Fetch play stats by play                | College Football Data MCP Server |                             |                                                                                                                           |
| Play types                  | HTTP Request Tool       | Fetch play types                        | College Football Data MCP Server |                             |                                                                                                                           |
| Play by play data            | HTTP Request Tool       | Fetch play-by-play data                 | College Football Data MCP Server |                             |                                                                                                                           |
| Description - plays          | Sticky Note             | Description label for Plays block      |                               |                              | ## 📋 Plays - Play by play data                                                                                           |
| Sticky Note8                  | Sticky Note             | Label for Metrics block                 |                               |                              | ## Metrics                                                                                                                |
| Win probability chart data   | HTTP Request Tool       | Fetch win probability chart data       | College Football Data MCP Server |                             |                                                                                                                           |
| Pregame win probability data | HTTP Request Tool       | Fetch pregame win probability data     | College Football Data MCP Server |                             |                                                                                                                           |
| Team Predicated Points Added (PPA/EPA) by game | HTTP Request Tool       | Fetch PPA/EPA by team and game         | College Football Data MCP Server |                             |                                                                                                                           |
| Player Predicated Points Added broken down by game | HTTP Request Tool       | Fetch PPA/EPA for player by game       | College Football Data MCP Server |                             |                                                                                                                           |
| Player Predicated Points Added broken down by season | HTTP Request Tool       | Fetch PPA/EPA for player by season     | College Football Data MCP Server |                             |                                                                                                                           |
| Predicted Points (Expected Points or EP) | HTTP Request Tool       | Fetch predicted points by down and distance | College Football Data MCP Server |                             |                                                                                                                           |
| Predicted Points Added (PPA/EPA) data by team | HTTP Request Tool       | Fetch PPA/EPA predicted points by team | College Football Data MCP Server |                             |                                                                                                                           |
| Description - metrics       | Sticky Note             | Description label for Metrics block    |                               |                              | ## 📋 Metrics - Data relating to Predicted Points and other metrics                                                      |
| Sticky Note9                  | Sticky Note             | Label for Players block                 |                               |                              | ## Players                                                                                                                |
| Transfer portal by season    | HTTP Request Tool       | Fetch transfer portal data              | College Football Data MCP Server |                             |                                                                                                                           |
| Team returning production metrics | HTTP Request Tool       | Fetch team returning production data   | College Football Data MCP Server |                             |                                                                                                                           |
| Search for player information | HTTP Request Tool       | Search player info by term and filters | College Football Data MCP Server |                             |                                                                                                                           |
| Player usage metrics broken down by season | HTTP Request Tool       | Fetch player usage metrics per season  | College Football Data MCP Server |                             |                                                                                                                           |
| Player stats by season       | HTTP Request Tool       | Fetch player stats for season           | College Football Data MCP Server |                             |                                                                                                                           |
| Description - players       | Sticky Note             | Description label for Players block    |                               |                              | ## 📋 Players - Player information and data                                                                             |
| Sticky Note10                 | Sticky Note             | Label for Rankings block                |                               |                              | ## Rankings                                                                                                              |
| Historical polls and rankings | HTTP Request Tool       | Fetch historical polls and rankings    | College Football Data MCP Server |                             |                                                                                                                           |
| Description - rankings      | Sticky Note             | Description label for Rankings block   |                               |                              | ## 📋 Rankings - Historical poll rankings                                                                                |
| Sticky Note11                 | Sticky Note             | Label for Ratings block                 |                               |                              | ## Ratings                                                                                                               |
| Historical Elo ratings      | HTTP Request Tool       | Fetch historical Elo ratings            | College Football Data MCP Server |                             |                                                                                                                           |
| Historical SP+ ratings      | HTTP Request Tool       | Fetch historical SP+ ratings            | College Football Data MCP Server |                             |                                                                                                                           |
| Historical SP+ ratings by conference | HTTP Request Tool       | Fetch SP+ ratings by conference         | College Football Data MCP Server |                             |                                                                                                                           |
| Historical SRS ratings      | HTTP Request Tool       | Fetch historical SRS ratings            | College Football Data MCP Server |                             |                                                                                                                           |
| Description - ratings      | Sticky Note             | Description label for Ratings block    |                               |                              | ## 📋 Ratings - Team rating data                                                                                          |
| Sticky Note12                | Sticky Note             | Label for Recruiting block              |                               |                              | ## Recruiting                                                                                                            |
| Recruit position group ratings | HTTP Request Tool       | Fetch recruiting position group ratings | College Football Data MCP Server |                             |                                                                                                                           |
| Player recruiting ratings and rankings | HTTP Request Tool       | Fetch player recruiting ratings         | College Football Data MCP Server |                             |                                                                                                                           |
| Team recruiting rankings and ratings | HTTP Request Tool       | Fetch team recruiting rankings          | College Football Data MCP Server |                             |                                                                                                                           |
| Description - recruiting    | Sticky Note             | Description label for Recruiting block |                               |                              | ## 📋 Recruiting - Recruiting rankings and data                                                                         |
| Sticky Note13                | Sticky Note             | Label for Teams block                   |                               |                              | ## Teams                                                                                                                 |
| Team rosters               | HTTP Request Tool       | Fetch team rosters                      | College Football Data MCP Server |                             |                                                                                                                           |
| Team talent composite rankings | HTTP Request Tool       | Fetch team talent composite rankings    | College Football Data MCP Server |                             |                                                                                                                           |
| Team information           | HTTP Request Tool       | Fetch team information                   | College Football Data MCP Server |                             |                                                                                                                           |
| FBS team list              | HTTP Request Tool       | Fetch FBS team list                      | College Football Data MCP Server |                             |                                                                                                                           |
| Team matchup history       | HTTP Request Tool       | Fetch historical team matchup data       | College Football Data MCP Server |                             |                                                                                                                           |
| Description - teams        | Sticky Note             | Description label for Teams block       |                               |                              | ## 📋 Teams - Team information                                                                                           |
| Sticky Note14               | Sticky Note             | Label for Stats block                   |                               |                              | ## Stats                                                                                                                |
| Team stat categories       | HTTP Request Tool       | Fetch team stat categories               | College Football Data MCP Server |                             |                                                                                                                           |
| Advanced team metrics by game | HTTP Request Tool       | Fetch advanced team metrics by game     | College Football Data MCP Server |                             |                                                                                                                           |
| Team statistics by season  | HTTP Request Tool       | Fetch team statistics by season          | College Football Data MCP Server |                             |                                                                                                                           |
| Advanced team metrics by season | HTTP Request Tool       | Fetch advanced team metrics by season   | College Football Data MCP Server |                             |                                                                                                                           |
| Description - stats        | Sticky Note             | Description label for Stats block       |                               |                              | ## 📋 Stats - Statistical data                                                                                           |
| Sticky Note15              | Sticky Note             | Label for Venues block                  |                               |                              | ## Venues                                                                                                               |
| Arena and venue information | HTTP Request Tool       | Fetch arena and venue info                | College Football Data MCP Server |                             |                                                                                                                           |
| Description - venues       | Sticky Note             | Description label for Venues block      |                               |                              | ## 📋 Venues - Information about venues                                                                                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create MCP Trigger Node:**  
   - Type: `@n8n/n8n-nodes-langchain.mcpTrigger`  
   - Set path to `college-football-data-mcp`  
   - Authentication: HTTP Header Auth with key name `Authorization`  
   - Create and assign API Key credential with value `Bearer your_key` (replace with your actual key)  

2. **Create HTTP Request Tool Nodes for Each API Endpoint:**  
   For each endpoint below, create an HTTP Request Tool node with the following template:  
   - Authentication: Use the same HTTP Header Auth credential as MCP Trigger  
   - URL: Set to the respective API endpoint URL (see below)  
   - Method: GET (default)  
   - Query Parameters: Add parameters as specified, setting each value to `$fromAI('parameterName', 'description', 'type', 'defaultValue')` if applicable  
   - Tool Description: Add the description from the workflow  

   **Endpoints to create:**  
   - Season calendar: `/calendar`  
   - Advanced box scores: `/game/box/advanced`  
   - Games and results: `/games`  
   - Game media information and schedules: `/games/media`  
   - Player game stats: `/games/players`  
   - Team game stats: `/games/teams`  
   - Game weather information (Patreon only): `/games/weather`  
   - Team records: `/records`  
   - Live game results (Patreon only): `/scoreboard`  
   - Coaching records and history: `/coaches`  
   - Conferences: `/conferences`  
   - List of NFL Draft picks: `/draft/picks`  
   - List of NFL positions: `/draft/positions`  
   - List of NFL teams: `/draft/teams`  
   - Drive data and results: `/drives`  
   - Betting lines: `/lines`  
   - Live metrics and PBP (Patreon only): `/live/plays`  
   - Types of player play stats: `/play/stat/types`  
   - Play stats by play: `/play/stats`  
   - Play types: `/play/types`  
   - Play by play data: `/plays`  
   - Win probability chart data: `/metrics/wp`  
   - Pregame win probability data: `/metrics/wp/pregame`  
   - Team Predicated Points Added (PPA/EPA) by game: `/ppa/games`  
   - Player Predicated Points Added by game: `/ppa/players/games`  
   - Player Predicated Points Added by season: `/ppa/players/season`  
   - Predicted Points (Expected Points or EP): `/ppa/predicted`  
   - Predicted Points Added (PPA/EPA) data by team: `/ppa/teams`  
   - Transfer portal by season: `/player/portal`  
   - Team returning production metrics: `/player/returning`  
   - Search for player information: `/player/search`  
   - Player usage metrics broken down by season: `/player/usage`  
   - Player stats by season: `/stats/player/season`  
   - Historical polls and rankings: `/rankings`  
   - Historical Elo ratings: `/ratings/elo`  
   - Historical SP+ ratings: `/ratings/sp`  
   - Historical SP+ ratings by conference: `/ratings/sp/conferences`  
   - Historical SRS ratings: `/ratings/srs`  
   - Recruit position group ratings: `/recruiting/groups`  
   - Player recruiting ratings and rankings: `/recruiting/players`  
   - Team recruiting rankings and ratings: `/recruiting/teams`  
   - Team rosters: `/roster`  
   - Team talent composite rankings: `/talent`  
   - Team information: `/teams`  
   - FBS team list: `/teams/fbs`  
   - Team matchup history: `/teams/matchup`  
   - Team stat categories: `/stats/categories`  
   - Advanced team metrics by game: `/stats/game/advanced`  
   - Team statistics by season: `/stats/season`  
   - Advanced team metrics by season: `/stats/season/advanced`  
   - Arena and venue information: `/venues`  

3. **Connect Each HTTP Request Tool Node to the MCP Trigger:**  
   - For each HTTP Request Tool node, create an output connection from the MCP Trigger node's `ai_tool` output to the HTTP Request Tool node's input.  
   - This setup allows the MCP Trigger to selectively invoke the appropriate endpoint based on the AI agent's request.

4. **Add Sticky Notes for Documentation and Organization:**  
   - Add sticky notes as labels for each thematic block (Games, Coaches, Conferences, etc.)  
   - Add descriptive sticky notes explaining each block's purpose  

5. **Configure Credentials:**  
   - Create an HTTP Header Auth credential with the header key `Authorization`  
   - Set the value to your CollegeFootballData.com API key prefixed with `Bearer ` as required  
   - Assign this credential to all HTTP Request Tool nodes and MCP Trigger node  

6. **Set Defaults and Constraints:**  
   - Use default values in `$fromAI()` expressions where applicable (e.g., `seasonType` defaults to `regular`)  
   - Mark parameters as optional or required according to API documentation  
   - Ensure numeric parameters are handled as numbers, strings as strings, booleans as booleans  

7. **Activate the Workflow:**  
   - Once all nodes are created, connected, and configured, activate the workflow  
   - Retrieve the webhook URL from the MCP Trigger node for use by AI agents  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                 | Context or Link                                                                                                                  |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------|
| API keys must be supplied with "Bearer " prefix, e.g., "Bearer your_key". Keys are obtainable from the CollegeFootballData.com website.                                                                                                                                                    | Authentication instructions                                                                                                     |
| The workflow exposes 51 distinct endpoints covering comprehensive college football data including games, coaches, conferences, draft info, betting, plays, metrics, players, rankings, ratings, recruiting, teams, stats, and venues. AI parameters use `$fromAI()` expressions for dynamic query construction. | Overall workflow capability description                                                                                         |
| Patreon-only endpoints (e.g., game weather, live game results, live metrics) require appropriate subscription access and may return errors if unauthorized.                                                                                                                                  | Special access notes                                                                                                            |
| For MCP integration details and customization guidance, refer to the official n8n documentation and join the Discord channel at https://discord.me/cfomodz                                                                                                                                | Support and customization resources                                                                                            |
| Consider adding data transformation, error handling, logging, and monitoring nodes as needed for production readiness.                                                                                                                                                                      | Suggested workflow enhancements                                                                                                |

---

**Disclaimer:** The provided text is exclusively derived from a workflow designed with n8n, respecting all current content policies and containing no illegal or protected content. All data processed is legal and publicly available.