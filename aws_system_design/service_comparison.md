# AWS Services Comparison

This document provides a comprehensive comparison between various AWS services commonly used in system design.

## Database Services

### RDS Aurora vs DynamoDB

| Feature | Amazon Aurora | Amazon DynamoDB |
|---------|---------------|-----------------|
| **Type** | Relational Database (MySQL/PostgreSQL compatible) | NoSQL, key-value and document database |
| **Scalability** | Vertical scaling for writes, read replicas for read scaling | Seamless horizontal scaling for both reads and writes |
| **Consistency** | ACID compliant | Eventually consistent by default, with option for strong consistency |
| **Use Cases** | Complex queries, transactions, joins | High-throughput applications, serverless apps, microservices |
| **Performance** | 5x throughput of standard MySQL, 3x of PostgreSQL | Single-digit millisecond latency at any scale |
| **Pricing Model** | Hourly instance charges + storage | Pay-per-request or provisioned capacity |
| **Maintenance** | Lower maintenance than standard RDS | Fully managed, zero maintenance |
| **Backup/Recovery** | Automated backups, point-in-time recovery | Continuous backups with point-in-time recovery |
| **Global Distribution** | Global database option with cross-region replication | Global tables with multi-region replication |
| **Schema Flexibility** | Fixed schema, requires migrations | Flexible schema, no migrations needed |

**When to choose Aurora:**
- Applications requiring complex SQL queries, joins, and transactions
- When migrating from existing MySQL/PostgreSQL databases
- Applications with predictable workloads and read-heavy scenarios
- When strong consistency and ACID properties are required

**When to choose DynamoDB:**
- Serverless applications with unpredictable traffic patterns
- Applications requiring consistent single-digit millisecond latency
- Microservices requiring a simple API for data access
- IoT, gaming, and other high-throughput applications
- When schema flexibility and minimal operational overhead are priorities

## Compute Services

### EC2 vs Lambda vs ECS vs Fargate

| Feature | EC2 | Lambda | ECS (EC2 launch type) | ECS with Fargate |
|---------|-----|--------|----------------------|------------------|
| **Provisioning** | Manual instance provisioning | Serverless, no provisioning | Requires EC2 instance management | Serverless, no instance management |
| **Scaling** | Auto Scaling groups | Automatic scaling | Cluster Auto Scaling | Automatic scaling |
| **Payment Model** | Pay for instance uptime | Pay per invocation and duration | Pay for EC2 instances | Pay per task execution time |
| **Execution Duration** | Unlimited | Up to 15 minutes | Unlimited | Unlimited |
| **Management Overhead** | High (OS patching, etc.) | Very low | Medium (clusters, not instances) | Low |
| **Use Cases** | Traditional apps, custom runtime | Event-driven functions, microservices | Containerized applications | Containerized apps without infrastructure management |
| **Cold Start** | Warm-up required | Cold start latency | No cold start for running instances | Cold start latency |
| **Memory Limits** | Up to 24TB (u-*) | Up to 10GB | Based on instance type | Up to 120GB |
| **CPU Limits** | Up to 448 vCPUs | Limited control (tied to memory) | Based on instance type | Up to 16 vCPUs |

**When to choose EC2:**
- Legacy applications requiring specific OS/runtime configurations
- Applications requiring GPUs or specialized hardware
- Workloads needing predictable performance and cost
- Applications requiring more than 15 minutes of runtime

**When to choose Lambda:**
- Event-driven workloads
- Microservices with short execution times
- APIs with variable traffic patterns
- Batch processing workloads that can be parallelized
- When operational simplicity is a priority

**When to choose ECS (EC2 launch type):**
- Containerized applications requiring fine-grained control
- Batch processing workloads with predictable patterns
- When you want to maximize resource utilization across applications
- Applications requiring specific EC2 instance types

**When to choose ECS with Fargate:**
- Containerized applications when you want to avoid cluster management
- Variable workloads that benefit from precise per-task billing
- Dev/test environments needing quick setup and minimal management
- Batch processing with unpredictable schedules

## Storage Services

### S3 vs EBS vs EFS

| Feature | S3 | EBS | EFS |
|---------|-----|-----|-----|
| **Type** | Object storage | Block storage | Network file system |
| **Access** | HTTP/HTTPS APIs | Attached to a single EC2 instance | Multiple EC2 instances simultaneously |
| **Use Cases** | Static files, backups, data lakes | Boot volumes, databases | Shared file systems, content management |
| **Durability** | 99.999999999% (11 9's) | 99.8% - 99.9% | 99.999999999% (11 9's) |
| **Availability** | 99.99% | Tied to instance availability | 99.99% (standard) or 99.9% (One Zone) |
| **Scalability** | Virtually unlimited | 1GB to 64TB per volume | Automatically scales to petabytes |
| **Performance** | Latency in ms, high throughput | Low latency, high IOPS | Higher latency than EBS, scales with size |
| **Cost** | Pay for storage, requests, data transfer | Pay for provisioned capacity | Pay for storage used |
| **Cross-AZ** | Built-in redundancy | Requires snapshots | Standard mode is multi-AZ |

**When to choose S3:**
- Static file hosting (websites, media, documents)
- Backup and disaster recovery
- Data lakes and big data analytics
- Content distribution (with CloudFront)
- When direct HTTP access is required

**When to choose EBS:**
- Boot volumes for EC2 instances
- Databases requiring high IOPS and low latency
- Applications requiring block-level storage
- Development environments requiring quick snapshots
- When data persistence independent of instance lifecycle is needed

**When to choose EFS:**
- Shared file systems accessed by multiple instances
- Content management systems
- Development environments
- Web serving and analytics applications
- Container storage
- Machine learning training and inference

## Caching Services

### ElastiCache (Redis) vs DAX vs CloudFront

| Feature | ElastiCache (Redis) | DynamoDB Accelerator (DAX) | CloudFront |
|---------|---------------------|----------------------------|------------|
| **Type** | In-memory data store | DynamoDB-specific cache | Content Delivery Network |
| **Use Cases** | General-purpose caching, session store | Accelerating DynamoDB access | Static and dynamic content delivery |
| **Complexity** | Moderate (requires cache management) | Low (works seamlessly with DynamoDB) | Low (configuration-based) |
| **Flexibility** | Highly flexible (data structures, Lua) | Limited to DynamoDB operations | HTTP/HTTPS content only |
| **Coverage** | Within VPC | Within VPC | Global edge locations |
| **Invalidation** | Programmatic | Automatic (TTL-based) | API-driven invalidation |
| **Latency Reduction** | Microseconds within region | Single-digit millisecond | Edge location proximity to users |

**When to choose ElastiCache (Redis):**
- General-purpose caching requirements
- Session storage
- Real-time leaderboards and counting
- Pub/sub messaging systems
- When advanced data structures are needed

**When to choose DAX:**
- Applications already using DynamoDB
- Read-heavy workloads on DynamoDB
- Repeated reads of the same items
- Applications requiring the lowest possible DynamoDB latency
- When you want to avoid application-level caching logic

**When to choose CloudFront:**
- Global user base requiring low-latency access
- Static asset delivery (images, CSS, JavaScript)
- Dynamic content that can be cached
- Security (with AWS Shield and WAF integration)
- When you need to reduce origin load

## Messaging Services

### SQS vs SNS vs EventBridge vs Kinesis

| Feature | SQS | SNS | EventBridge | Kinesis Data Streams |
|---------|-----|-----|-------------|---------------------|
| **Type** | Queue | Pub/Sub | Event bus | Streaming data |
| **Message Retention** | Up to 14 days | No retention | No retention | 24 hours to 365 days |
| **Consumers** | Single consumer per message | Multiple subscribers | Multiple targets | Multiple consumers |
| **Ordering** | FIFO available | No ordering guarantees | No ordering guarantees | Partition-level ordering |
| **Throughput** | Virtually unlimited | Virtually unlimited | Virtually unlimited | Provisioned (by shard) |
| **Use Cases** | Decoupling services | Fanout notifications | Event-driven architectures | Real-time analytics |
| **Message Size** | Up to 256KB | Up to 256KB | Up to 256KB | Up to 1MB |
| **Delivery** | At-least-once (standard), Exactly-once (FIFO) | At-least-once | At-least-once | At-least-once |

**When to choose SQS:**
- Decoupling microservices
- Handling traffic spikes with buffered processing
- Distributing tasks among multiple workers
- When message ordering (FIFO) or exactly-once processing is required
- Background job processing

**When to choose SNS:**
- Fanout to multiple subscribers
- Push notifications to mobile devices
- Email and SMS notifications
- When you need to broadcast messages to multiple services

**When to choose EventBridge:**
- Event-driven architectures
- Integration with SaaS applications
- Complex event filtering and routing
- Scheduled events (cron jobs)
- When you need content-based filtering

**When to choose Kinesis Data Streams:**
- Real-time analytics
- Log and event data collection
- Metrics and monitoring data
- Click streams and application logs
- IoT data processing
- When you need to process high-volume streaming data with ordering guarantees

## API Gateway Services

### API Gateway vs Application Load Balancer

| Feature | API Gateway | Application Load Balancer |
|---------|-------------|----------------------------|
| **Type** | Managed API Gateway | Layer 7 Load Balancer |
| **Use Cases** | API management, serverless APIs | HTTP/HTTPS application balancing |
| **Features** | Authorization, throttling, caching, API keys | TLS termination, path-based routing, health checks |
| **Integration** | Lambda, HTTP, AWS services | EC2, ECS, Lambda, IP addresses |
| **Pricing** | Pay per request | Pay per hour + data processed |
| **Scalability** | Fully managed, automatic | Fully managed, automatic |
| **Deployment** | Stages (dev, prod) | No built-in staging |
| **Monitoring** | CloudWatch, X-Ray integration | CloudWatch integration |

**When to choose API Gateway:**
- Building RESTful or WebSocket APIs
- Serverless applications
- When you need API management features (throttling, keys, documentation)
- Microservices architectures with complex routing requirements
- When you need to expose AWS services directly as APIs

**When to choose Application Load Balancer:**
- Traditional web applications
- Container-based applications
- When you need path-based routing to microservices
- WebSocket applications not requiring API management
- When you primarily need HTTP/HTTPS load balancing without advanced API features

## Serverless Application Services

### Lambda vs Step Functions vs AppSync

| Feature | Lambda | Step Functions | AppSync |
|---------|--------|---------------|---------|
| **Type** | Function as a Service | Workflow orchestration | GraphQL service |
| **Use Cases** | Event-driven processing | Complex workflows, state machines | Real-time data synchronization, APIs |
| **Execution Time** | Up to 15 minutes | Up to 1 year | Based on underlying resolvers |
| **State Management** | Stateless | Stateful | GraphQL schema defines state |
| **Integration** | Many event sources | AWS services, HTTP APIs | DynamoDB, Lambda, HTTP, RDS |
| **Development** | Function code | State machine definition | GraphQL schema, resolvers |
| **Complexity** | Low | Medium | Medium to high |

**When to choose Lambda:**
- Simple, discrete functions
- Event-driven processing
- Microservices
- Real-time file processing
- Backend for web and mobile applications

**When to choose Step Functions:**
- Complex, multi-step workflows
- Error handling and retry logic
- Long-running processes
- Service orchestration
- When visual workflow representation is valuable

**When to choose AppSync:**
- Real-time applications
- Offline data synchronization
- Mobile and web applications requiring GraphQL
- When you need subscriptions and real-time updates
- When you want to combine data from multiple sources