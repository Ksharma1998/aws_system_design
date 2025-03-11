# System Design: URL Shortening Service

## Problem Statement
Design a scalable URL shortening service similar to TinyURL or Bitly that can:
- Convert long URLs to short, unique aliases
- Redirect users from shortened URLs to original URLs
- Provide analytics on URL usage
- Handle high traffic volume (millions of redirects per day)
- Ensure high availability and low latency

## Requirements Analysis

### Functional Requirements
- URL shortening: Convert a long URL to a short URL
- URL redirection: Redirect from short URL to original URL
- Custom aliases: Allow users to choose custom short URLs
- URL expiration: Set expiration dates for links
- Analytics: Track click rates, geographic data, referrers, etc.
- User accounts: Optional accounts for managing URLs

### Non-Functional Requirements
- High availability (99.9%+)
- Low latency redirects (< 100ms)
- Scalability to handle millions of redirects per day
- Persistent storage of URLs
- Security to prevent abuse

## Architecture Overview

This solution uses a serverless architecture on AWS with the following components:

1. **API Layer** - Handles URL shortening and redirection requests
2. **Storage Layer** - Stores URL mappings and metadata
3. **Analytics Layer** - Processes and stores usage data
4. **Cache Layer** - Caches frequent redirects for performance

## Architecture Diagram

```
┌─────────────┐     ┌───────────────┐     ┌────────────────┐
│    Users    │────▶│  CloudFront   │────▶│  API Gateway   │
└─────────────┘     └───────────────┘     └────────┬───────┘
                                                   │
                 ┌────────────────────────────────┬┴────────────────────┐
                 ▼                                ▼                     ▼
          ┌─────────────┐                 ┌─────────────┐       ┌─────────────┐
          │  Shortening │                 │ Redirection │       │  Analytics  │
          │   Lambda    │                 │   Lambda    │       │   Lambda    │
          └──────┬──────┘                 └──────┬──────┘       └──────┬──────┘
                 │                               │                     │
                 │                               ▼                     │
                 │                        ┌─────────────┐              │
                 │                        │ ElastiCache │              │
                 │                        │   (Redis)   │              │
                 │                        └──────┬──────┘              │
                 │                               │                     │
                 ▼                               ▼                     ▼
          ┌─────────────┐                 ┌─────────────┐       ┌─────────────┐
          │  DynamoDB   │◀────────────────│  DynamoDB   │       │  Kinesis    │
          │(User Data)  │                 │(URL Mapping)│       │Data Firehose│
          └─────────────┘                 └─────────────┘       └──────┬──────┘
                                                                       │
                                                                       ▼
                                                                ┌─────────────┐
                                                                │     S3      │
                                                                │(Click Data) │
                                                                └──────┬──────┘
                                                                       │
                                                                       ▼
                                                                ┌─────────────┐
                                                                │   Athena    │
                                                                └──────┬──────┘
                                                                       │
                                                                       ▼
                                                                ┌─────────────┐
                                                                │ QuickSight  │
                                                                └─────────────┘
```

## AWS Services Selection and Rationale

### API Layer
- **Amazon CloudFront**: For global content delivery and caching
  - **Why CloudFront?** Reduces latency for users worldwide and provides edge caching
  - **Decision point**: Using CloudFront over direct API Gateway access reduces latency by ~30-100ms depending on user location

- **Amazon API Gateway**: Manages API endpoints for shortening and redirection
  - **Why API Gateway over ALB?** API Gateway integrates directly with Lambda and provides API management features

### Compute Layer
- **AWS Lambda**: For URL shortening, redirection, and analytics processing
  - **Why Lambda over EC2/ECS?** 
    - URL shortening is highly suitable for serverless due to its stateless nature
    - Unpredictable traffic patterns benefit from Lambda's automatic scaling
    - No need to manage servers for simple request/response workflows
  - **Decision point**: Lambda's cost model works well for bursty traffic patterns typical of URL shorteners

### Storage Layer
- **Amazon DynamoDB**: Primary database for URL mappings and user data
  - **Why DynamoDB over Aurora/RDS?** 
    - URL redirection requires consistent single-digit millisecond latency
    - Simple key-value access pattern is ideal for NoSQL
    - Automatic scaling handles unpredictable traffic without manual intervention
  - **Decision point**: The URL mapping table uses a composite key structure with hash key as the short URL and sort key as the creation date

- **Amazon ElastiCache (Redis)**: Caches frequently accessed URLs
  - **Why Redis caching?** Reduces DynamoDB read costs and improves latency
  - **Decision point**: Redis was chosen over DAX (DynamoDB Accelerator) for its flexibility and cost-effectiveness for this use case

### Analytics Layer
- **Amazon Kinesis Data Firehose**: Streams click data for processing
  - **Why Kinesis Firehose?** Buffers and delivers click data to S3 without managing infrastructure
  - **Decision point**: Firehose chosen over Kinesis Data Streams due to simpler implementation and no need for real-time processing

- **Amazon S3**: Stores click analytics data
  - **Why S3 for analytics storage?** Cost-effective for large volumes of semi-structured data

- **Amazon Athena**: SQL queries on click data
  - **Why Athena over Redshift?** Serverless query service with no infrastructure to manage
  - **Decision point**: Athena chosen for its ability to query data directly in S3 without ETL processes

- **Amazon QuickSight**: Visualization dashboards for URL analytics
  - **Decision point**: QuickSight integrates seamlessly with Athena and provides interactive dashboards

## Data Model

### DynamoDB Schema

**URL Table**:
- `short_id` (Partition Key): The short URL identifier
- `original_url`: The original long URL
- `creation_date`: When the URL was shortened
- `expiration_date`: When the URL expires (optional)
- `user_id`: Owner of the URL (optional)
- `custom_alias`: Whether this is a custom alias
- `status`: Active or inactive

**User Table**:
- `user_id` (Partition Key): Unique user identifier
- `email`: User's email
- `tier`: Free or premium
- `creation_date`: Account creation date
- `url_count`: Number of URLs created

## URL Shortening Algorithm

Two approaches were considered:

1. **Counter-based approach**: Use an incrementing counter and convert to base62
   - **Pros**: Short URLs, sequential
   - **Cons**: Reveals information about URL creation order, requires a central counter

2. **Hash-based approach**: Hash the long URL and take first few characters
   - **Pros**: Distributed generation, no central counter
   - **Cons**: Potential collisions, longer URLs

**Decision point**: We selected a hybrid approach:
- Generate a random 6-character string using base62 encoding (a-z, A-Z, 0-9)
- Check for collisions in DynamoDB
- If collision occurs, regenerate or append a character

This gives 62^6 = ~57 billion possible combinations with very low collision probability.

## Scaling and Reliability Considerations

### Read vs. Write Scaling
- Write operations (URL creation): Much less frequent than reads
- Read operations (URL redirection): High volume, requiring aggressive caching

### Caching Strategy
- Redis TTL-based caching for frequent URLs
- Cache eviction policy: LRU (Least Recently Used)
- Cache hit ratio target: >95%

### DynamoDB Capacity
- On-demand capacity mode for unpredictable traffic
- For very high scale: Switch to provisioned capacity with auto-scaling

### Regional Deployment
- Multi-region deployment for reduced latency and higher availability
- DynamoDB global tables for data replication

## Security Considerations

- **Rate Limiting**: Prevent abuse through API Gateway throttling
- **Link Scanning**: Lambda function to check destination URL against malware databases
- **Data Encryption**: Encryption at rest and in transit
- **HTTPS**: Enforce HTTPS for all requests

## Analytics Implementation

- **Click Tracking**: Capture timestamp, IP (anonymized), user-agent, referer
- **Aggregations**: Hourly, daily, weekly click metrics
- **Geographic Data**: IP to location mapping for geographic insights
- **Custom Campaigns**: UTM parameter tracking for marketing campaigns

## Cost Optimization Strategies

- **Lambda**: Optimize memory allocation based on observed performance
- **DynamoDB**: Implement TTL for automatic deletion of expired URLs
- **Caching**: Aggressive caching to reduce DynamoDB read costs
- **CloudFront**: Edge caching for most popular redirects

## Implementation Plan

1. **Phase 1**: Core URL shortening and redirection functionality
2. **Phase 2**: User accounts and custom aliases
3. **Phase 3**: Analytics dashboard and reporting
4. **Phase 4**: API for integrations and advanced features

## Monitoring and Operations

- **CloudWatch Alarms**: Monitor redirect latency, error rates
- **X-Ray**: Trace request flow through the system
- **Dashboard**: Operational metrics for system health
- **Logs**: Centralized logging with CloudWatch Logs

## Comparison of Alternative Approaches

### Serverless vs. Container-based
- **Serverless Approach** (selected): Lower operational overhead, pay-per-use model
- **Container-based**: More control, potentially lower cost at consistent high volume
- **Decision point**: Serverless chosen for simplicity and ability to scale to zero

### SQL vs. NoSQL
- **NoSQL** (selected): Better scalability for simple key-value lookups
- **SQL Database**: Better for complex queries and relationships
- **Decision point**: NoSQL chosen due to simple data access patterns and latency requirements