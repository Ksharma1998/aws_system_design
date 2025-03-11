# System Design: Real-time Chat Application

## Problem Statement
Design a scalable, reliable, and real-time chat application similar to WhatsApp or Slack that can:
- Support millions of users with low-latency messaging
- Allow one-on-one and group conversations
- Provide message synchronization across multiple devices
- Support media sharing (images, videos, documents)
- Implement read receipts and typing indicators
- Ensure message delivery even when recipients are offline
- Maintain message history and search functionality

## Requirements Analysis

### Functional Requirements
- User registration and authentication
- User profile management and status
- Contact/friend management
- One-on-one messaging
- Group chat creation and management
- Message delivery with receipts (sent, delivered, read)
- Typing indicators
- Offline message delivery
- Media sharing (images, videos, documents)
- Message history and search
- Push notifications

### Non-Functional Requirements
- Low latency message delivery (< 100ms in normal conditions)
- High availability (99.9%+)
- Scalability to millions of concurrent users
- Data consistency across multiple devices
- Message persistence and durability
- End-to-end encryption for privacy
- Multi-device synchronization

## Architecture Overview

The solution uses a combination of serverless and container-based services on AWS with the following components:

1. **Connection Service** - Manages WebSocket connections for real-time messaging
2. **Message Service** - Processes and routes messages
3. **User Service** - Manages user profiles and authentication
4. **Group Service** - Handles group chat metadata and membership
5. **Media Service** - Processes and stores media attachments
6. **Notification Service** - Delivers push notifications
7. **Search Service** - Indexes and searches message history

## Architecture Diagram

```
┌───────────┐      ┌───────────────┐      ┌────────────────┐
│  Clients  │─────▶│  API Gateway  │─────▶│ Application    │
└───────────┘      │  (WebSocket)  │      │ Load Balancer  │
     │             └───────────────┘      └────────┬───────┘
     │                     │                       │
     │                     ▼                       │
     │            ┌───────────────┐                │
     │            │  Connection   │                │
     │            │   Lambda      │                │
     │            └───────┬───────┘                │
     │                    │                        │
     │                    ▼                        │
     │            ┌───────────────┐                │
     │            │  DynamoDB     │                │
     │            │ (Connections) │                │
     │            └───────────────┘                │
     │                                             │
     └─────────────────┬─────────────┬────────────┘
                       │             │
                       ▼             ▼
              ┌────────────────┐    ┌────────────────┐
              │ Message Service│    │  User Service  │
              │    (ECS)       │    │     (ECS)      │
              └────────┬───────┘    └────────┬───────┘
                       │                     │
                       ▼                     ▼
              ┌────────────────┐    ┌────────────────┐
              │  ElastiCache   │    │    DynamoDB    │
              │    (Redis)     │    │   (User Data)  │
              └────────┬───────┘    └────────────────┘
                       │
                       ▼
              ┌────────────────┐    ┌────────────────┐    ┌────────────────┐
              │    DynamoDB    │    │  Media Service │    │ Search Service │
              │  (Messages)    │    │    (Lambda)    │    │ (OpenSearch)   │
              └────────────────┘    └────────┬───────┘    └────────┬───────┘
                                             │                     │
                                             ▼                     ▼
                                    ┌────────────────┐    ┌────────────────┐
                                    │       S3       │    │  OpenSearch    │
                                    │(Media Storage) │    │   Service      │
                                    └────────┬───────┘    └────────────────┘
                                             │
                                             ▼
                                    ┌────────────────┐    ┌────────────────┐
                                    │   CloudFront   │    │  Notification  │
                                    │                │    │  Lambda        │
                                    └────────────────┘    └────────┬───────┘
                                                                   │
                                                                   ▼
                                                          ┌────────────────┐
                                                          │  Amazon SNS    │
                                                          │                │
                                                          └────────────────┘
```

## AWS Services Selection and Rationale

### Connection Management
- **Amazon API Gateway (WebSocket)**: Manages WebSocket connections for real-time communication
  - **Why WebSocket API Gateway?** Provides managed WebSocket connections with built-in scaling
  - **Decision point**: WebSocket API was chosen over HTTP polling for true real-time communication with lower latency and less bandwidth consumption

- **AWS Lambda (Connection Handler)**: Processes WebSocket connection events
  - **Why Lambda for connections?** Serverless scaling handles connection spikes efficiently
  - **Decision point**: Using Lambda for connection events provides cost efficiency for the connection management layer

- **Amazon DynamoDB (Connection Table)**: Tracks active connections and their metadata
  - **Why DynamoDB for connections?** Fast lookups by user ID and connection ID with millisecond latency
  - **Decision point**: TTL feature automatically removes stale connections

### Message Processing
- **Amazon ECS with Fargate**: Runs the message processing service
  - **Why ECS/Fargate over Lambda?** Message routing logic may exceed Lambda's 15-minute timeout for complex operations
  - **Decision point**: Fargate chosen over EC2 for reduced management overhead while maintaining container flexibility

- **Amazon ElastiCache (Redis)**: Caches user presence information and recent messages
  - **Why Redis?** Provides pub/sub capabilities perfect for real-time communication between server instances
  - **Decision point**: Redis chosen over other caching solutions for its Pub/Sub feature which enables real-time updates

- **Amazon DynamoDB (Message Table)**: Stores message history
  - **Why DynamoDB for messages?** 
    - Consistent performance at scale for write-heavy workloads
    - Flexible schema for different message types
    - DynamoDB Streams enables real-time processing of new messages
  - **Decision point**: Single-table design with GSIs to support various access patterns

### User Management
- **Amazon Cognito**: Handles user authentication and identity
  - **Why Cognito?** Provides secure authentication with social identity federation
  - **Decision point**: Cognito User Pools manages the user directory with MFA support

- **Amazon ECS with Fargate**: Hosts the user service
  - **Decision point**: Containerized approach allows for complex user management logic not suitable for Lambda

- **Amazon DynamoDB (User Table)**: Stores user profiles and metadata
  - **Decision point**: User information requires strong consistency for profile updates

### Media Handling
- **AWS Lambda**: Processes uploaded media
  - **Why Lambda for media processing?** Well-suited for event-driven file processing
  - **Decision point**: Lambda's integration with S3 events makes it ideal for processing new media uploads

- **Amazon S3**: Stores media files
  - **Why S3?** Durability, availability, and cost-effectiveness for object storage
  - **Decision point**: S3 lifecycle policies move older media to lower-cost storage classes

- **Amazon CloudFront**: Delivers media content
  - **Why CloudFront?** Reduces latency for media delivery and offloads origin
  - **Decision point**: CloudFront's global edge network ensures low-latency media access worldwide

### Search Functionality
- **Amazon OpenSearch Service**: Indexes and searches messages
  - **Why OpenSearch over DynamoDB queries?** Full-text search capabilities essential for message content
  - **Decision point**: OpenSearch's text analysis and relevance scoring provide better search results than simple contains queries

### Notifications
- **AWS Lambda**: Processes notification events
  - **Why Lambda for notifications?** Event-driven model matches notification workflows

- **Amazon SNS**: Delivers push notifications to mobile devices
  - **Why SNS?** Managed service for sending notifications to iOS, Android, and other platforms
  - **Decision point**: SNS chosen over custom notification solution for its multi-platform support

## Data Model

### DynamoDB Schema

**Connection Table**:
- `connection_id` (Partition Key): WebSocket connection ID
- `user_id`: User identifier
- `connected_at`: Connection timestamp
- `device_info`: Device metadata
- `status`: Online/Away/Busy
- `ttl`: Auto-expiration for abandoned connections

**Message Table** (single-table design):
- `conversation_id` (Partition Key): One-on-one or group conversation ID
- `message_id` (Sort Key): Timestamp-based message ID
- `sender_id`: ID of the sending user
- `content`: Message content (for text messages)
- `media_url`: Link to media content (if any)
- `media_type`: Type of media attachment
- `status`: Sent/Delivered/Read
- `read_by`: Set of user IDs who have read the message
- `client_id`: Client-generated message ID for deduplication

**User Table**:
- `user_id` (Partition Key): User identifier
- `email`: User's email address
- `name`: User's display name
- `profile_pic`: URL to profile picture
- `status`: Custom status message
- `last_seen`: Last online timestamp
- `devices`: List of registered devices

**Group Table**:
- `group_id` (Partition Key): Group identifier
- `name`: Group name
- `created_by`: User who created the group
- `created_at`: Creation timestamp
- `members`: Set of member user IDs
- `admins`: Set of admin user IDs
- `avatar`: Group avatar URL

## Real-time Messaging Implementation

### Message Flow
1. User sends message via WebSocket connection
2. Connection Lambda validates and routes to Message Service
3. Message Service:
   - Stores message in DynamoDB
   - Publishes to Redis channels for online recipients
   - Triggers notifications for offline recipients
4. For online recipients, messages are pushed via WebSocket
5. For offline recipients, messages are queued for delivery when they reconnect

### Delivery Guarantees
- **At-least-once delivery**: Messages are retried until acknowledgment
- **Message ordering**: Timestamp-based ordering within conversations
- **Offline delivery**: Messages stored persistently, delivered on reconnection

## Scaling and Reliability Considerations

### Connection Scaling
- WebSocket API Gateway automatically scales to handle millions of connections
- Connection state stored in DynamoDB, not in memory
- Regional deployment with global routing for failure resilience

### Message Processing Scaling
- Message Service scales horizontally with ECS tasks
- Redis pub/sub clusters for high-throughput messaging
- DynamoDB auto-scaling for message storage

### Data Access Patterns
- Recent messages cached in Redis for fast access
- Historical messages retrieved from DynamoDB
- Access patterns optimized with GSIs for conversation-based queries

## Security Considerations

- **End-to-End Encryption**: Messages encrypted on client side before transmission
- **In-Transit Encryption**: TLS for all connections
- **At-Rest Encryption**: DynamoDB and S3 encryption
- **Access Control**: IAM roles and policies for service-to-service communication
- **Authentication**: Cognito with MFA for user authentication

## Multi-Device Synchronization

- **Message Sync**: New device fetches recent history from DynamoDB
- **Read State Sync**: Read receipts propagated to all user devices
- **Presence Sync**: User presence information shared across devices

## Cost Optimization Strategies

- **Connection Management**: WebSocket connections closed after inactivity period
- **Message Storage**: Tiered storage approach with hot/cold separation
- **Media Processing**: Resize and compress media on upload
- **Database Design**: Careful key design to minimize query costs
- **Caching Strategy**: Redis caching to reduce DynamoDB reads

## Monitoring and Analytics

- **CloudWatch**: Operational metrics and alarms
- **X-Ray**: Distributed tracing for message flow
- **ElastiCache metrics**: Monitor Redis performance
- **Custom dashboards**: Track message volume, delivery times, and user activity

## Implementation Considerations

### Group Chat Optimizations
- Fan-out messages only to online group members
- Batch updates for large groups
- Rate limiting for very active groups

### Media Handling Strategy
- Progressive loading for large media files
- Thumbnail generation for quick previews
- Transcoding to optimize for different devices

### Typing Indicators
- Throttled WebSocket messages for typing status
- Expiration for "typing" state to handle disconnections

## Alternative Design Considerations

### MQTT vs. WebSockets
- **WebSockets** (selected): Wider browser support, simpler implementation
- **MQTT**: More efficient for very constrained devices
- **Decision point**: WebSockets chosen for broader compatibility and simplicity

### Lambda vs. Long-Running Services
- **Lambda** (for connections): Serverless scaling, pay-per-use
- **ECS** (for message processing): More suitable for complex, stateful processing
- **Decision point**: Hybrid approach leverages strengths of both models