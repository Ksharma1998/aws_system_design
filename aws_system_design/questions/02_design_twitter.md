# System Design: Twitter-like Social Media Platform

## Problem Statement
Design a scalable, high-performance social media platform similar to Twitter that can:
- Support hundreds of millions of users
- Allow users to post short messages (tweets)
- Support following other users
- Generate a personalized timeline/feed for each user
- Provide search functionality
- Support media attachments
- Handle high write and read traffic

## Requirements Analysis

### Functional Requirements
- User registration and authentication
- Creating, reading, updating, and deleting tweets
- Following/unfollowing users
- User timeline (tweets from followed users)
- Home timeline (personalized feed)
- Search functionality (hashtags, content, users)
- Media uploads (images, videos)
- Notifications for mentions and interactions

### Non-Functional Requirements
- High availability (99.99%)
- Low latency for timeline generation and tweet posting
- Horizontal scalability for both reads and writes
- Eventually consistent model is acceptable
- Real-time features for notifications

## Architecture Overview

This solution uses a microservices architecture on AWS with the following components:

1. **User Service** - Manages user accounts and relationships
2. **Tweet Service** - Handles creating and retrieving tweets
3. **Timeline Service** - Generates and delivers personalized timelines
4. **Media Service** - Manages media uploads and processing
5. **Search Service** - Provides search functionality
6. **Notification Service** - Handles real-time notifications

## Architecture Diagram

```
┌────────────┐     ┌──────────────┐     ┌───────────────┐
│   Users    │────▶│  CloudFront  │────▶│ Application   │
└────────────┘     └──────────────┘     │ Load Balancer │
                                        └───────┬───────┘
                                                │
                                                ▼
                                        ┌───────────────┐
                                        │  API Gateway  │
                                        └───────┬───────┘
         ┌───────────────┬─────────────────────┼─────────────┬────────────────┐
         ▼               ▼                     ▼             ▼                ▼
┌────────────────┐ ┌─────────────┐     ┌─────────────┐ ┌──────────────┐ ┌────────────┐
│  User Service  │ │Tweet Service│     │   Timeline  │ │ Media Service │ │   Search   │
│   (ECS/Fargate)│ │ (ECS/Fargate)│     │   Service   │ │ (ECS/Fargate) │ │  Service   │
└───────┬────────┘ └──────┬──────┘     │ (ECS/Fargate)│ └───────┬──────┘ │(ECS/Fargate)│
        │                 │            └───────┬─────┘         │        └──────┬─────┘
        ▼                 ▼                    ▼               ▼               ▼
┌────────────────┐ ┌─────────────┐     ┌─────────────┐  ┌─────────────┐ ┌────────────┐
│  DynamoDB      │ │  DynamoDB   │     │  ElastiCache │  │     S3      │ │OpenSearch  │
│  (User Data)   │ │(Tweet Data) │     │   (Redis)    │  │  (Media)    │ │ Service    │
└────────────────┘ └─────────────┘     └─────────────┘  └─────────────┘ └────────────┘
        │                 │                   ▲
        └────────────────┼───────────────────┘
                         │
                         ▼
                  ┌─────────────┐     ┌─────────────────┐
                  │   Kinesis   │────▶│Lambda (Analytics)│
                  │Data Streams │     └─────────────────┘
                  └──────┬──────┘              │
                         │                     ▼
                         ▼               ┌─────────────┐
                  ┌─────────────┐        │   Amazon    │
                  │ EventBridge │        │  Redshift   │
                  └──────┬──────┘        └─────────────┘
                         │
                         ▼
                ┌──────────────────┐     ┌───────────────┐
                │ Notification Svc │────▶│   Amazon SNS  │
                │   (Lambda)       │     └───────────────┘
                └──────────────────┘
```

## AWS Services Selection and Rationale

### API Layer
- **Amazon API Gateway**: Chosen to manage and secure the API endpoints
  - **Why API Gateway over ALB?** API Gateway provides additional features like throttling, authentication, and request/response transformation

- **Amazon CloudFront**: For content delivery and static asset caching
  - **Decision point**: CloudFront was chosen over directly exposing the API to reduce latency and provide DDoS protection

### Compute Layer
- **Amazon ECS with Fargate**: For running the microservices
  - **Why ECS with Fargate over EC2?** Fargate eliminates the need to manage servers while providing container orchestration
  - **Why not Lambda?** Some services like the Timeline service have longer processing times and higher memory requirements that make containers more appropriate
  - **Decision point**: ECS was chosen over Kubernetes (EKS) for its simpler management and tighter integration with AWS

### Data Storage
- **Amazon DynamoDB**: Primary database for user and tweet data
  - **Why DynamoDB over Aurora?** Twitter-like workloads require extreme scale and low-latency access patterns. DynamoDB provides:
    - Consistent single-digit millisecond performance at any scale
    - Automatic scaling for both reads and writes
    - Global tables for multi-region redundancy
  - **Decision point**: We chose DynamoDB's On-Demand capacity mode for unpredictable workloads with traffic spikes

- **Amazon ElastiCache (Redis)**: For caching timelines and reducing database load
  - **Why Redis over Memcached?** Redis provides data structures (sorted sets) perfect for timeline implementation
  - **Decision point**: We use Redis sorted sets to store user timelines, with tweet IDs as members and timestamps as scores

- **Amazon S3**: For storing media files (images, videos)
  - **Decision point**: S3 Standard for recent media, transitioning to S3 Infrequent Access for older content

### Search Functionality
- **Amazon OpenSearch Service**: For search functionality
  - **Why OpenSearch over building on DynamoDB?** OpenSearch provides advanced full-text search capabilities including fuzzy matching and relevance scoring
  - **Decision point**: OpenSearch provides more sophisticated search features than simple string comparisons in DynamoDB

### Real-time Features
- **Amazon Kinesis Data Streams**: For processing activity streams in real-time
  - **Why Kinesis over SQS?** Kinesis maintains order and allows multiple consumers, important for activity feeds

- **Amazon EventBridge**: For event-driven architecture components
  - **Decision point**: EventBridge was chosen over direct service-to-service communication for better decoupling and event filtering

- **AWS Lambda**: For real-time processing and notifications
  - **Why Lambda here but not for the main services?** Lambda is perfect for event-triggered processing like notifications

### Notifications
- **Amazon SNS**: For delivering push notifications
  - **Why SNS over a custom notification system?** SNS handles the complexity of delivering to multiple platforms (iOS, Android, web)

### Analytics
- **Amazon Redshift**: For data warehousing and analytics
  - **Decision point**: Redshift was chosen for analytics over Athena because of the high-volume, structured nature of the data and need for complex queries

## Scaling and Reliability Considerations

### Data Access Patterns
- **Timeline Generation**: Pre-computed and cached in Redis to avoid computing on read
  - **Why this approach?** Computing timelines on-the-fly doesn't scale with millions of users

- **Tweet Fanout**: When a user tweets, the tweet ID is pushed to all follower timelines in Redis
  - **Decision point**: For users with millions of followers (celebrities), fanout is handled differently using a hybrid approach

### Read vs. Write Scaling
- DynamoDB global secondary indexes separate read and write paths
- Read replicas in ElastiCache improve read throughput

### High Availability
- Multi-AZ deployment for all services
- DynamoDB global tables for multi-region redundancy
- Auto-scaling for ECS tasks based on CPU/memory utilization

## Security Considerations

- **Authentication**: Amazon Cognito for user authentication
- **Authorization**: IAM roles for service-to-service communication
- **Data Encryption**: Encryption at rest and in transit
- **DDoS Protection**: AWS Shield with CloudFront

## Cost Optimization Strategies

- **DynamoDB Reserved Capacity**: For predictable base load
- **Auto-scaling**: Scale resources based on demand
- **S3 Intelligent Tiering**: For media storage cost optimization
- **Caching Strategy**: Reduce database reads with ElastiCache

## Implementation Considerations

### Timeline Generation Approaches
1. **Pull Model**: Fetch tweets from followed users on demand (doesn't scale)
2. **Push Model**: Fan out tweets to follower timelines (our approach for most users)
3. **Hybrid Model**: Push for users with few followers, pull for celebrities with millions of followers

### Handling Viral Tweets
- Rate limiting at API Gateway prevents abuse
- Auto-scaling handles viral content surges
- Caching popular tweets reduces database load

### Data Consistency
- Eventually consistent model for timeline updates
- Strong consistency for user profile operations

## Monitoring and Analytics

- **CloudWatch**: Monitor service metrics and set alarms
- **X-Ray**: Distributed tracing across microservices
- **CloudTrail**: Audit API calls for security analysis
- **Redshift**: Analytics on user behavior and platform usage