# Architecture Plan for Doconomy-AWorld Integration

## 1. System Overview

This integration will provide a light gamification layer through personalized content (stories, quizzes, tips) for Mastercard users, with the following key characteristics:

- High-volume, low-latency API (millions of requests daily, sub-200ms response time)
- Stateless design where Doconomy/bank maintains the authoritative user state
- Each API call includes the complete user state
- Minimal data storage on AWorld side (user IDs and anonymized interaction data for reporting purposes)
- OAuth2 authentication
- Cloud-native microservices architecture using Node.js/TypeScript

```mermaid
flowchart TB
    User[End User] -->|Interacts with| MastercardApp[Mastercard App/Website]
    MastercardApp -->|Calls| DoconomyAPI[Doconomy API]
    DoconomyAPI -->|Calls with user state| AWorldAPI[AWorld API]
    AWorldAPI -->|Returns content/results| DoconomyAPI
    DoconomyAPI -->|Updates UI| MastercardApp
    
    AWorldAPI -->|Logs minimal data| AWorldDB[(AWorld DB)]
    DoconomyAPI -->|Stores complete user state| DoconomyDB[(Doconomy/Bank DB)]
    
    subgraph "AWorld System"
        AWorldAPI
        AWorldDB
    end
    
    subgraph "Doconomy/Mastercard System"
        DoconomyAPI
        DoconomyDB
        MastercardApp
    end
```

## 2. API Design

### 2.1 API Endpoints

AWorld will expose the following RESTful API endpoints:

1. **Personalized Story Delivery**
   - `GET /api/v1/stories`
   - Provides personalized stories/tips based on user state

2. **Story Completion Submission**
   - `POST /api/v1/stories/{storyId}/answers`
   - Processes quiz answers and returns results

3. **Quiz Delivery**
   - `GET /api/v1/quizzes`
   - Delivers appropriate quizzes based on user state

4. **Quiz Answer Submission**
   - `POST /api/v1/quizzes/{quizId}/answers`
   - Processes quiz answers and returns results

### 2.2 Request/Response Format

All API requests will follow a similar pattern:

```mermaid
sequenceDiagram
    participant D as Doconomy
    participant A as AWorld
    
    D->>A: POST /api/v1/[endpoint]
    Note right of D: Request includes:<br/>- Authentication<br/>- Complete user state<br/>- Endpoint-specific parameters
    
    A->>A: Process request<br/>Log minimal interaction data
    
    A->>D: Response with requested content/confirmation
    Note left of A: Response includes:<br/>- Requested content<br/>- Status information<br/>- Updated user state (when applicable)
```

**Sample Request Format:**

```json
{
  "userState": {
    "userId": "user123",
    "preferences": { ... },
    "completedQuizzes": [ ... ],
    "viewedStories": [ ... ],
    "exp": 1250,
    "level": 3,
    ...
  },
  "requestParams": {
    // Endpoint-specific parameters
    ...
  }
}
```

**Sample Response Format:**

```json
{
  "status": "success",
  "data": {
    // Endpoint-specific response data
    ...
  },
  "updatedUserState": {
    // Updated user state (when applicable)
    "userId": "user123",
    "exp": 1300,  // Updated from 1250
    "level": 3,
    "completedQuizzes": [ ... ],  // With newly completed quiz
    ...
  }
}
```

## 3. Data Storage Model

AWorld will maintain storage of anonymized data for reporting and analytics purposes:

```mermaid
erDiagram
    User ||--o{ Interaction : has
    
    User {
        string userId
        timestamp createdAt
    }
    
    Interaction {
        string interactionId
        string userId
        string contentId
        string interactionType
        timestamp timestamp
        json metadata
    }
```

- **User**: Stores only the user ID and creation timestamp
- **Interaction**: Records anonymized interaction data (content views, quiz completions, etc.)

This storage is separate from the authoritative user state maintained by Doconomy/bank but allows AWorld to generate reports and analytics on content performance and user engagement patterns.

## 4. Synchronous Gamification Updates

The primary approach uses synchronous gamification updates where all logic (exp, levels, etc.) is processed immediately:

```mermaid
sequenceDiagram
    participant User
    participant Bank
    participant Doconomy
    participant AWorld
    
    User->>Bank: Takes action (completes quiz)
    Bank->>Doconomy: Forwards action
    Doconomy->>AWorld: API call with current user state
    Note right of AWorld: AWorld processes quiz<br/>AND updates gamification<br/>(exp, levels, achievements)<br/>Stores anonymized data
    AWorld->>Doconomy: Response with content and fully updated user state
    Doconomy->>Doconomy: Stores fully updated user state
    Doconomy->>Bank: Updates UI with all changes
    Bank->>User: Shows content/results and gamification updates
    
    Note over User,AWorld: All updates are immediate, no asynchronous updates needed
```

This approach couples the business logic of different domains (quiz processing and gamification) but provides immediate feedback to users about their progress. The trade-off is increased complexity in the AWorld system, which now needs to handle all gamification logic synchronously.

## 5. Alternative Approach for Large Content Libraries

### Plan B: Minimal State Storage on Doconomy/Bank Side

If the bank decides to offer a very large content library (e.g., thousands of stories, quizzes, etc.), storing all interaction history in the user state on Doconomy/bank side could become problematic. In this scenario, an alternative approach would be:

```mermaid
flowchart TB
    User[End User] -->|Interacts with| MastercardApp[Mastercard App/Website]
    MastercardApp -->|Calls| DoconomyAPI[Doconomy API]
    DoconomyAPI -->|Calls with minimal user state| AWorldAPI[AWorld API]
    AWorldAPI -->|Returns content/results| DoconomyAPI
    DoconomyAPI -->|Updates UI| MastercardApp
    
    AWorldAPI -->|Stores complete interaction history| AWorldDB[(AWorld DB)]
    DoconomyAPI -->|Stores minimal user state| DoconomyDB[(Doconomy/Bank DB)]
    
    subgraph "AWorld System"
        AWorldAPI
        AWorldDB
    end
    
    subgraph "Doconomy/Mastercard System"
        DoconomyAPI
        DoconomyDB
        MastercardApp
    end
```

In this approach:

1. Doconomy/bank stores only essential user state:
   - User ID
   - Current level
   - Total experience points
   - Other minimal gamification metrics

2. AWorld stores the complete interaction history:
   - All viewed stories
   - All completed quizzes with answers
   - Detailed interaction timestamps
   - Content recommendations history

```mermaid
sequenceDiagram
    participant User
    participant Bank
    participant Doconomy
    participant AWorld
    
    User->>Bank: Takes action (completes quiz)
    Bank->>Doconomy: Forwards action
    Doconomy->>AWorld: API call with minimal user state
    Note right of AWorld: AWorld processes quiz<br/>Updates gamification<br/>Stores complete interaction history
    AWorld->>Doconomy: Response with content and updated minimal state
    Doconomy->>Doconomy: Stores minimal user state
    Doconomy->>Bank: Updates UI
    Bank->>User: Shows content/results
    
    Note over User,AWorld: Later: User requests content recommendations
    
    User->>Bank: Requests content
    Bank->>Doconomy: Forwards request
    Doconomy->>AWorld: API call with minimal user state + userId
    Note right of AWorld: AWorld looks up complete<br/>interaction history by userId<br/>to generate recommendations
    AWorld->>Doconomy: Response with personalized content
    Doconomy->>Bank: Updates UI
    Bank->>User: Shows personalized content
```

This approach is recommended only if the content library is expected to grow significantly large, as it shifts more responsibility to the AWorld system.
