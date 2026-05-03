Full Instagram API MCP Server

https://n8nworkflows.xyz/workflows/full-instagram-api-mcp-server-5581


# Full Instagram API MCP Server

### 1. Workflow Overview

This workflow, titled **Full Instagram API MCP Server**, is designed to provide a comprehensive backend server interface for interacting with Instagram’s API through an MCP (Multi-Channel Platform) Trigger. It serves as a centralized API server handling multiple Instagram API endpoints, enabling users to perform various Instagram data retrieval and interaction tasks programmatically via HTTP requests.

The key use cases include fetching media by location or tag, managing media comments and likes, searching users and tags, and handling user relationships. The workflow is structured into logical blocks based on Instagram API resource types and operations.

**Logical Blocks:**

- **1.1 MCP Server Trigger (Entry Point):** Receives API requests via the MCP trigger.
- **1.2 Media Retrieval by Location:** Queries Instagram media by geographic coordinates, location details, or area.
- **1.3 Media Retrieval by Popularity and Shortcode:** Fetches popular media, media by shortcode, and related media details.
- **1.4 Media Comments Management:** Retrieves, creates, and deletes comments on media.
- **1.5 Media Likes Management:** Manages media likes including liking and unliking media, and fetching likes.
- **1.6 Tag Search and Media by Tag:** Searches tags by name and retrieves media associated with tags.
- **1.7 User Search and User Media:** Searches users, gets user feeds, liked media, details, and recent media.
- **1.8 User Relationship Management:** Manages followers, following, follow requests, and relationships between users.

---

### 2. Block-by-Block Analysis

#### 1.1 MCP Server Trigger (Entry Point)

- **Overview:**  
  This is the workflow entry point configured as an MCP trigger node that listens for incoming API requests and routes them to the appropriate Instagram API request nodes.

- **Nodes Involved:**  
  - Instagram MCP Server

- **Node Details:**  

  - **Instagram MCP Server**  
    - Type: MCP Trigger (from Langchain n8n nodes)  
    - Configuration: Listens with a specific webhook ID for incoming requests representing different Instagram API commands.  
    - Key Variables: Acts as the single API gateway, expects parameters that define the desired Instagram API action.  
    - Input: External API call or HTTP request.  
    - Output: Routes to various HTTP request nodes based on the Instagram API method requested.  
    - Failure modes: Webhook timeout, malformed request data, or missing parameters could cause failures.

#### 1.2 Media Retrieval by Location

- **Overview:**  
  This block allows querying recent media based on geographic information like coordinates and location IDs.

- **Nodes Involved:**  
  - Get Recent Media by Geo  
  - Search Locations by Coordinates  
  - Get Location Details  
  - Get Recent Media by Location

- **Node Details:**  

  - **Get Recent Media by Geo**  
    - Type: HTTP Request  
    - Role: Fetches recent media from Instagram given geographic coordinates.  
    - Config: HTTP request to Instagram’s media search endpoint filtered by geo-location.  
    - Input: Routed from MCP Server with geo-coordinates parameters.  
    - Output: Media items data.  
    - Edge cases: Invalid coordinates, API rate limits, or empty results.

  - **Search Locations by Coordinates**  
    - Type: HTTP Request  
    - Role: Finds Instagram location IDs from given latitude/longitude.  
    - Config: Calls Instagram’s location search endpoint.  
    - Input: Coordinates from trigger.  
    - Output: Location IDs for further queries.  
    - Edge cases: No locations found, invalid parameters.

  - **Get Location Details**  
    - Type: HTTP Request  
    - Role: Retrieves detailed metadata of a specific Instagram location by ID.  
    - Input: Location ID from previous node.  
    - Output: Location metadata.  
    - Edge cases: Invalid location ID, permissions.

  - **Get Recent Media by Location**  
    - Type: HTTP Request  
    - Role: Fetches recent media posted at a specific location.  
    - Input: Location ID.  
    - Output: Media list.  
    - Edge cases: No media found, location deprecated.

#### 1.3 Media Retrieval by Popularity and Shortcode

- **Overview:**  
  Retrieves popular media and media specified by shortcode, including detailed information.

- **Nodes Involved:**  
  - Get Popular Media  
  - Search Media by Area  
  - Get Media by Shortcode  
  - Get Media Details

- **Node Details:**  

  - **Get Popular Media**  
    - Type: HTTP Request  
    - Role: Retrieves currently popular media on Instagram.  
    - Input: Trigger parameters.  
    - Output: List of popular media.  
    - Edge cases: API limit, empty results.

  - **Search Media by Area**  
    - Type: HTTP Request  
    - Role: Searches media within a defined geographical area.  
    - Input: Area parameters.  
    - Output: Media list.  
    - Edge cases: Area invalid or no media.

  - **Get Media by Shortcode**  
    - Type: HTTP Request  
    - Role: Fetches media details using Instagram shortcode identifier.  
    - Input: Shortcode string.  
    - Output: Media metadata.  
    - Edge cases: Invalid shortcode, media deleted.

  - **Get Media Details**  
    - Type: HTTP Request  
    - Role: Retrieves detailed information about a specific media item.  
    - Input: Media ID or shortcode.  
    - Output: Full media details.  
    - Edge cases: Missing media or access restrictions.

#### 1.4 Media Comments Management

- **Overview:**  
  Handles retrieval, creation, and deletion of comments on Instagram media.

- **Nodes Involved:**  
  - Get Media Comments  
  - Create Media Comment  
  - Delete Media Comment

- **Node Details:**  

  - **Get Media Comments**  
    - Type: HTTP Request  
    - Role: Retrieves comments on a given media item.  
    - Input: Media ID.  
    - Output: List of comments.  
    - Edge cases: No comments, permissions.

  - **Create Media Comment**  
    - Type: HTTP Request  
    - Role: Posts a new comment on specified media.  
    - Input: Media ID and comment text.  
    - Output: Confirmation or new comment data.  
    - Edge cases: Spam filters, API rate limits.

  - **Delete Media Comment**  
    - Type: HTTP Request  
    - Role: Deletes a comment from media.  
    - Input: Comment ID and media ID.  
    - Output: Confirmation of deletion.  
    - Edge cases: Comment not found, insufficient permissions.

#### 1.5 Media Likes Management

- **Overview:**  
  Manages liking and unliking media and retrieving who liked a media item.

- **Nodes Involved:**  
  - Like Media  
  - Remove Media Like  
  - Get Media Likes

- **Node Details:**  

  - **Like Media**  
    - Type: HTTP Request  
    - Role: Likes a media item on behalf of the user.  
    - Input: Media ID.  
    - Output: Success confirmation.  
    - Edge cases: Already liked, auth errors.

  - **Remove Media Like**  
    - Type: HTTP Request  
    - Role: Removes a like from media.  
    - Input: Media ID.  
    - Output: Confirmation.  
    - Edge cases: Like not found, permissions.

  - **Get Media Likes**  
    - Type: HTTP Request  
    - Role: Retrieves list of users who liked media.  
    - Input: Media ID.  
    - Output: User list.  
    - Edge cases: No likes, API limits.

#### 1.6 Tag Search and Media by Tag

- **Overview:**  
  Enables searching tags by name and retrieving media associated with tags.

- **Nodes Involved:**  
  - Search Tags by Name  
  - Get Tag Details  
  - Get Recent Media by Tag

- **Node Details:**  

  - **Search Tags by Name**  
    - Type: HTTP Request  
    - Role: Finds Instagram tags matching a search string.  
    - Input: Tag name.  
    - Output: Matching tags.  
    - Edge cases: No matches.

  - **Get Tag Details**  
    - Type: HTTP Request  
    - Role: Retrieves metadata about a specific tag.  
    - Input: Tag ID or name.  
    - Output: Tag details.  
    - Edge cases: Tag deleted or private.

  - **Get Recent Media by Tag**  
    - Type: HTTP Request  
    - Role: Fetches recent media tagged with a specific tag.  
    - Input: Tag ID/name.  
    - Output: Media list.  
    - Edge cases: No media, tag deprecated.

#### 1.7 User Search and User Media

- **Overview:**  
  Provides functionality to search users and retrieve their media and profile information.

- **Nodes Involved:**  
  - Search Users by Name  
  - Get User Feed  
  - Get User Liked Media  
  - Get User Details  
  - Get User Recent Media

- **Node Details:**  

  - **Search Users by Name**  
    - Type: HTTP Request  
    - Role: Searches for Instagram users by username or real name.  
    - Input: Search string.  
    - Output: User list.  
    - Edge cases: No results.

  - **Get User Feed**  
    - Type: HTTP Request  
    - Role: Retrieves posts on a user’s feed.  
    - Input: User ID.  
    - Output: Media list.  
    - Edge cases: Private account, no posts.

  - **Get User Liked Media**  
    - Type: HTTP Request  
    - Role: Fetches media that the user has liked.  
    - Input: User ID.  
    - Output: Media list.  
    - Edge cases: No likes, permissions.

  - **Get User Details**  
    - Type: HTTP Request  
    - Role: Retrieves profile information of a user.  
    - Input: User ID.  
    - Output: User data.  
    - Edge cases: User banned or private.

  - **Get User Recent Media**  
    - Type: HTTP Request  
    - Role: Gets recent media posted by the user.  
    - Input: User ID.  
    - Output: Media list.  
    - Edge cases: No recent media.

#### 1.8 User Relationship Management

- **Overview:**  
  Manages follow requests, followers, following lists, and user-to-user relationships.

- **Nodes Involved:**  
  - Get Follow Requests  
  - Get User Followers  
  - Get User Following  
  - Get User Relationship  
  - Update User Relationship

- **Node Details:**  

  - **Get Follow Requests**  
    - Type: HTTP Request  
    - Role: Retrieves pending follow requests for the authenticated user.  
    - Input: User token.  
    - Output: Request list.  
    - Edge cases: No requests pending.

  - **Get User Followers**  
    - Type: HTTP Request  
    - Role: Lists followers of a user.  
    - Input: User ID.  
    - Output: Followers list.  
    - Edge cases: Private accounts.

  - **Get User Following**  
    - Type: HTTP Request  
    - Role: Lists users that a user is following.  
    - Input: User ID.  
    - Output: Following list.  
    - Edge cases: Private or restricted.

  - **Get User Relationship**  
    - Type: HTTP Request  
    - Role: Retrieves the relationship status between two users (e.g. following, followed by).  
    - Input: User IDs.  
    - Output: Relationship status.  
    - Edge cases: Blocked users.

  - **Update User Relationship**  
    - Type: HTTP Request  
    - Role: Updates relationship status (follow/unfollow/block).  
    - Input: User ID and desired action.  
    - Output: Confirmation.  
    - Edge cases: API restrictions, errors on invalid actions.

---

### 3. Summary Table

| Node Name                 | Node Type                       | Functional Role                    | Input Node(s)            | Output Node(s)                | Sticky Note                              |
|---------------------------|--------------------------------|----------------------------------|-------------------------|------------------------------|-----------------------------------------|
| Setup Instructions        | Sticky Note                    | Workflow general info            | -                       | -                            |                                         |
| Workflow Overview         | Sticky Note                    | Workflow general info            | -                       | -                            |                                         |
| Instagram MCP Server      | MCP Trigger                   | API Gateway/Trigger              | -                       | Multiple HTTP Request nodes   |                                         |
| Sticky Note               | Sticky Note                    | Context notes                   | -                       | -                            |                                         |
| Get Recent Media by Geo   | HTTP Request                  | Fetch media by geo coordinates   | Instagram MCP Server     | -                            |                                         |
| Sticky Note2              | Sticky Note                    | Context notes                   | -                       | -                            |                                         |
| Search Locations by Coordinates | HTTP Request            | Search locations by lat/lon      | Instagram MCP Server     | Get Location Details          |                                         |
| Get Location Details      | HTTP Request                  | Get metadata for location        | Search Locations by Coordinates | Get Recent Media by Location |                                         |
| Get Recent Media by Location | HTTP Request                | Fetch media by location          | Get Location Details     | -                            |                                         |
| Sticky Note3              | Sticky Note                    | Context notes                   | -                       | -                            |                                         |
| Get Popular Media         | HTTP Request                  | Fetch popular media              | Instagram MCP Server     | -                            |                                         |
| Search Media by Area      | HTTP Request                  | Search media by area             | Instagram MCP Server     | -                            |                                         |
| Get Media by Shortcode    | HTTP Request                  | Fetch media by shortcode        | Instagram MCP Server     | Get Media Details            |                                         |
| Get Media Details         | HTTP Request                  | Fetch detailed media info       | Get Media by Shortcode   | -                            |                                         |
| Sticky Note4              | Sticky Note                    | Context notes                   | -                       | -                            |                                         |
| Get Media Comments        | HTTP Request                  | Retrieve comments on media      | Instagram MCP Server     | -                            |                                         |
| Create Media Comment      | HTTP Request                  | Post comment on media           | Instagram MCP Server     | -                            |                                         |
| Delete Media Comment      | HTTP Request                  | Delete comment on media         | Instagram MCP Server     | -                            |                                         |
| Description - comments    | Sticky Note                    | Comments block description      | -                       | -                            |                                         |
| Sticky Note5              | Sticky Note                    | Context notes                   | -                       | -                            |                                         |
| Remove Media Like         | HTTP Request                  | Remove a like from media        | Instagram MCP Server     | -                            |                                         |
| Get Media Likes           | HTTP Request                  | Get users who liked media       | Instagram MCP Server     | -                            |                                         |
| Like Media                | HTTP Request                  | Like a media item               | Instagram MCP Server     | -                            |                                         |
| Description - likes       | Sticky Note                    | Likes block description         | -                       | -                            |                                         |
| Sticky Note6              | Sticky Note                    | Context notes                   | -                       | -                            |                                         |
| Search Tags by Name       | HTTP Request                  | Search tags by name             | Instagram MCP Server     | Get Tag Details              |                                         |
| Get Tag Details           | HTTP Request                  | Get metadata for tag            | Search Tags by Name      | Get Recent Media by Tag      |                                         |
| Get Recent Media by Tag   | HTTP Request                  | Get media with tag              | Get Tag Details          | -                            |                                         |
| Sticky Note7              | Sticky Note                    | Context notes                   | -                       | -                            |                                         |
| Search Users by Name      | HTTP Request                  | Search Instagram users          | Instagram MCP Server     | Get User Feed                |                                         |
| Get User Feed             | HTTP Request                  | Get user feed media             | Search Users by Name     | -                            |                                         |
| Get User Liked Media      | HTTP Request                  | Get media liked by user         | Search Users by Name     | -                            |                                         |
| Get User Details          | HTTP Request                  | Get user profile info           | Search Users by Name     | Get User Recent Media        |                                         |
| Get User Recent Media     | HTTP Request                  | Get recent media by user        | Get User Details         | -                            |                                         |
| Sticky Note8              | Sticky Note                    | Context notes                   | -                       | -                            |                                         |
| Get Follow Requests       | HTTP Request                  | Get pending follow requests     | Instagram MCP Server     | -                            |                                         |
| Get User Followers        | HTTP Request                  | List user followers             | Instagram MCP Server     | -                            |                                         |
| Get User Following        | HTTP Request                  | List users user is following    | Instagram MCP Server     | -                            |                                         |
| Get User Relationship     | HTTP Request                  | Get relationship status         | Instagram MCP Server     | -                            |                                         |
| Update User Relationship  | HTTP Request                  | Follow/unfollow/block user      | Instagram MCP Server     | -                            |                                         |
| Description - relationships | Sticky Note                  | Relationships block description | -                       | -                            |                                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create MCP Trigger Node**  
   - Add node: MCP Trigger (Langchain MCP Trigger)  
   - Set webhook ID for receiving API calls (unique webhook identifier)  
   - This node acts as the main entry point for all Instagram API requests.

2. **Create HTTP Request Nodes for Media Retrieval by Location**  
   - Add "Get Recent Media by Geo" HTTP Request node. Configure with Instagram API endpoint to fetch recent media by geo-coordinates.  
   - Add "Search Locations by Coordinates" HTTP Request node to get location IDs from lat/lon.  
   - Add "Get Location Details" HTTP Request node to fetch detailed location metadata using location ID.  
   - Add "Get Recent Media by Location" HTTP Request node to get media by location ID.  
   - Connect nodes in the sequence: MCP Trigger → Search Locations by Coordinates → Get Location Details → Get Recent Media by Location. Also connect MCP Trigger → Get Recent Media by Geo.

3. **Create HTTP Request Nodes for Media Retrieval by Popularity and Shortcode**  
   - Add "Get Popular Media" node with endpoint for popular media.  
   - Add "Search Media by Area" node for area-based media search.  
   - Add "Get Media by Shortcode" node to fetch media by shortcode.  
   - Add "Get Media Details" node to get detailed info on media.  
   - Connect MCP Trigger to these nodes accordingly, and connect Get Media by Shortcode → Get Media Details.

4. **Create HTTP Request Nodes for Media Comments Management**  
   - Add "Get Media Comments" node to retrieve comments.  
   - Add "Create Media Comment" node to post comments.  
   - Add "Delete Media Comment" node to delete comments.  
   - Connect all three nodes directly to MCP Trigger.

5. **Create HTTP Request Nodes for Media Likes Management**  
   - Add "Like Media" node to like a media item.  
   - Add "Remove Media Like" node to unlike a media item.  
   - Add "Get Media Likes" node to retrieve who liked a media.  
   - Connect all three nodes to MCP Trigger.

6. **Create HTTP Request Nodes for Tag Search and Media by Tag**  
   - Add "Search Tags by Name" node to search tags.  
   - Add "Get Tag Details" node for tag metadata.  
   - Add "Get Recent Media by Tag" node to fetch media by tag.  
   - Connect: MCP Trigger → Search Tags by Name → Get Tag Details → Get Recent Media by Tag.

7. **Create HTTP Request Nodes for User Search and Media**  
   - Add "Search Users by Name" node to search users.  
   - Add "Get User Feed" node for user’s feed.  
   - Add "Get User Liked Media" node for liked media.  
   - Add "Get User Details" node for user info.  
   - Add "Get User Recent Media" node for recent media.  
   - Connect: MCP Trigger → Search Users by Name → Get User Feed, Get User Liked Media, Get User Details → Get User Recent Media.

8. **Create HTTP Request Nodes for User Relationship Management**  
   - Add "Get Follow Requests" node.  
   - Add "Get User Followers" node.  
   - Add "Get User Following" node.  
   - Add "Get User Relationship" node.  
   - Add "Update User Relationship" node.  
   - Connect all nodes directly to MCP Trigger.

9. **Add Sticky Notes as Needed**  
   - Insert sticky notes at logical sections for instructions or descriptions.

10. **Configure Credentials**  
    - For each HTTP Request node, set up Instagram API credentials (OAuth2 or token-based as required).  
    - Ensure MCP Trigger node is configured with valid webhook and authentication.

11. **Set Default Parameter Values and Constraints**  
    - For each node, configure required query parameters, path variables, and headers.  
    - Validate inputs are sanitized and error handling is in place for API limits or invalid parameters.

---

### 5. General Notes & Resources

| Note Content                                                                 | Context or Link                          |
|------------------------------------------------------------------------------|-----------------------------------------|
| This workflow acts as a full MCP server for Instagram API integration, consolidating multiple Instagram endpoints behind a single webhook interface. | Workflow description                     |
| MCP Trigger node used is from the Langchain n8n nodes package for API orchestration. | Node type detail                         |
| When deploying, ensure you have Instagram API access with proper scopes and rate limits monitored. | Instagram API best practices             |
| Consider adding error handling nodes or retry logic in production for API rate limits or failures. | Best practice recommendation             |
| Sticky notes are used throughout the workflow for contextual instructions and descriptions, enhancing maintainability. | Workflow documentation approach          |

---

**Disclaimer:** The text provided is derived exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.