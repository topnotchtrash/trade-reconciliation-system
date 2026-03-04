# Trade Reconciliation System

> Event-driven financial pipeline simulator for validating trade processing at scale using synthetic data
[![Maven Central](https://img.shields.io/maven-central/v/io.github.topnotchtrash/annapurna.svg)](https://central.sonatype.com/artifact/io.github.topnotchtrash/annapurna)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)



## Overview

A distributed microservices system that simulates production-grade financial trade processing workflows using synthetic data. The system validates trade booking logic, infrastructure capacity, and reconciliation accuracy without touching production systems or real customer data.

### Key Features

- **High-Volume Processing**: Processes 100K+ synthetic trades end-to-end
- **Event-Driven Architecture**: Kafka-based message streaming for decoupled services
- **Polyglot Microservices**: Java (Spring Boot) + Python (FastAPI) services
- **Safe Execution**: Transaction rollback pattern prevents database pollution
- **Full Observability**: Prometheus metrics + Grafana dashboards
- **Production-Ready Patterns**: Multi-threading, consumer groups, async job tracking

---

## Tech Stack

| Layer | Technology | Purpose |
|-------|-----------|---------|
| **Data Generation** | Java 17, Maven | Synthetic trade creation |
| **Orchestration** | Spring Boot 3.2 | REST API + Kafka producer |
| **Workers** | Spring Boot 3.2 | Parallel trade processing |
| **Reconciliation** | Python 3.11, FastAPI | Results validation |
| **Messaging** | Apache Kafka | Event streaming |
| **Databases** | PostgreSQL, Redis | Shadow DB, job tracking |
| **Monitoring** | Prometheus, Grafana | Metrics & dashboards |
| **Containerization** | Docker, Docker Compose | Service orchestration |

---

## System Components

### 1. Annapurna (Synthetic Financial Trade Data Generator)

**Repository**: [github.com/topnotchtrash/annapurna](https://github.com/topnotchtrash/annapurna)  
**Maven Central**: [io.github.topnotchtrash:annapurna](https://central.sonatype.com/artifact/io.github.topnotchtrash/annapurna)

High-performance Java library for generating realistic synthetic financial trade data with proper business logic.

**Features**:
- Generates 5 trade types: Equity Swaps, Interest Rate Swaps, FX Forwards, Equity Options, Credit Default Swaps
- 3 data quality profiles for testing: CLEAN (valid data), EDGE_CASE (boundary conditions), STRESS (intentionally corrupted)
- Multi-threaded lock-free generation: 2.35M+ trades/second
- Configurable trade type distributions and data quality profiles
- JSON and CSV export support
- Zero PII/sensitive data

**Usage**:
```java
List<Trade> trades = Annapurna.builder()
    .tradeTypes()
        .equitySwap(30)
        .interestRateSwap(30)
        .fxForward(20)
        .option(15)
        .cds(5)
    .dataProfile()
        .clean(70)
        .edgeCase(20)
        .stress(10)
    .count(100000)
    .parallelism(8)
    .build()
    .generate();
```

**Technology**: Java 17, Maven, DataFaker, Jackson, JUnit

---

### 2. TradePublisher (Orchestration Service)

**Repository**: [github.com/topnotchtrash/trade-publisher](https://github.com/topnotchtrash/trade-publisher)

Spring Boot microservice that orchestrates synthetic trade generation and publishes to Kafka for downstream processing.

**Responsibilities**:
- REST API for triggering trade generation jobs
- Integrates Annapurna library for synthetic trade generation
- Publishes trades to Kafka in batches with partition-based load balancing
- Tracks async job progress in Redis with TTL-based expiration
- Exposes Prometheus metrics for monitoring

**API Endpoints**:
```bash
# Start reconciliation job
POST /api/reconciliation/start
{
  "tradeCount": 10000,
  "tradeDistribution": {
    "INTEREST_RATE_SWAP": 30,
    "EQUITY_SWAP": 30,
    "FX_FORWARD": 20,
    "EQUITY_OPTION": 15,
    "CREDIT_DEFAULT_SWAP": 5
  },
  "profileDistribution": {
    "CLEAN": 70,
    "EDGE_CASE": 20,
    "STRESS": 10
  }
}

# Get job status
GET /api/reconciliation/status/{jobId}

# Health check
GET /actuator/health

# Prometheus metrics
GET /actuator/prometheus
```

**Technology**: Java 17, Spring Boot 3.2.1, Spring Kafka, Spring Data Redis, Micrometer Prometheus, Maven, Docker

**Configuration**:
- Kafka topic: `trade-recon-input`
- Kafka partitions: 10 (partitioned by trade type)
- Redis TTL: 24 hours for job status
- Batch size: 100 trades per batch
- Async thread pool: 8 core threads, 16 max threads
- Port: 8089

**Performance**:
- 100K trades generated in ~30 seconds (3,300 trades/sec)
- Parallel batch publishing (200 batches/sec)
- Total job time: ~1 minute for 100K trades end-to-end

**Key Features**:
- Asynchronous job processing with @Async
- Multi-threaded trade generation using Annapurna parallelism
- Parallel Kafka publishing with CompletableFuture
- Redis-based job tracking with automatic expiration
- Partition-based message routing for consumer load balancing
- Comprehensive error handling and metrics instrumentation

---
```markdown
### 3. Sherpa (Worker Service)

**Repository**: [github.com/topnotchtrash/trade-forge](https://github.com/topnotchtrash/trade-forge)

Spring Boot microservice that consumes trades from Kafka and processes them in parallel.

**Responsibilities**:
- Consumes trades from Kafka via consumer groups
- Validates trade fields (required fields, date logic, counterparty)
- Applies business logic (pricing calculations, risk checks, spread adjustments)
- Writes to PostgreSQL inside transaction
- Rolls back transaction (simulation mode - no actual persistence)
- Emits processing metrics

**Processing Flow**:
```
1. Consume trade from Kafka
2. Validate fields (counterparty exists, dates valid, currency valid)
3. Enrich data (lookup rates, calculate spreads)
4. Apply business rules (pricing, P&L, risk limits)
5. Begin database transaction
6. Execute INSERT INTO trades (...)
7. ROLLBACK transaction (dry-run mode)
8. Log success/failure
9. Commit Kafka offset
```

**Technology**: Spring Boot, Spring Kafka, Spring Data JPA, PostgreSQL, CompletableFuture, Maven, Docker

**Scalability**:
- Runs 3 worker instances locally
- Kafka consumer group for load balancing
- Parallel processing with CompletableFuture
- Each worker processes trades independently

**Trade Processors**:
- `SwapProcessor.java` - Interest rate swap validation/pricing
- `EquitySwapProcessor.java` - Equity swap with spread calculation
- `FXForwardProcessor.java` - FX forward rate validation
- `OptionProcessor.java` - Equity option pricing logic
- `CDSProcessor.java` - Credit default swap processing

---

creating combined readme 
so is trade-forge info correct as of now if there any mistakes or any additions make them to above and give exactly in md so i can copy paste
```

### 4. TradeAudit (Pipeline Validation Service)

**Repository**: [github.com/topnotchtrash/tradeaudit](https://github.com/topnotchtrash/tradeaudit)

Python FastAPI service that validates end-to-end pipeline integrity by comparing expected vs actual trade processing results.

**Responsibilities**:
- Reads job metadata from Redis (expected trade counts, timing)
- Queries PostgreSQL shadow database for processed trades
- Compares expected vs actual results
- Calculates performance metrics (throughput, latency percentiles)
- Generates comprehensive validation reports
- Exposes results via REST API with smart timing logic

**Validation Logic**:
```python
# Get job metadata from Redis
job = redis_client.get_job(job_id)
expected_count = job["tradesGenerated"]
start_time = parse_timestamp(job["startTime"])

# Query PostgreSQL for processed trades
processed_trades = postgres_client.get_trades_since(start_time)

# Calculate discrepancies
summary = {
    "expected": expected_count,
    "processed": len(processed_trades),
    "successful": count_success(processed_trades),
    "failed": count_failed(processed_trades),
    "missing": expected_count - len(processed_trades),
    "successRate": (successful / expected) * 100
}
```

**API Endpoints**:
```bash
# Full validation report (with force parameter)
GET /api/tradeaudit/report/{jobId}?force=false

# Discrepancies only
GET /api/tradeaudit/discrepancies/{jobId}

# Quick job summary
GET /api/tradeaudit/summary/{jobId}

# Health check
GET /health
```

**Technology**: Python 3.11, FastAPI, SQLAlchemy, Pandas, PostgreSQL, Redis, Docker

**Features**:
- **Smart Timing**: Suggests waiting 5 minutes after job completion before generating report (allows Trade-Forge workers to finish processing)
- **Force Parameter**: Generate report immediately with warning if needed
- **Jackson JSON Parsing**: Handles Java serialization format from Trade-Publisher Redis storage

**Reports Generated**:
- Summary stats (expected vs processed, success rate, discrepancy rate)
- Performance metrics (throughput, latency p50/p95/p99/avg/max)
- Breakdown by trade type (total, successful, failed, success rate, avg latency)
- Failure analysis (total failed, failed by trade type, top 10 failures)
- Additional insights (top counterparties, top currencies with processing stats)
- Report metadata (generation timestamp, job completion time, elapsed time, forced flag, warnings)

---
### 5. Trade-Recon-Infra (Infrastructure as Code)

**Repository**: [github.com/topnotchtrash/trade-recon-infra](https://github.com/topnotchtrash/trade-recon-infra)

Centralized infrastructure orchestration using Terraform and Docker Compose. Provisions all AWS resources and provides local development environment for the trade reconciliation system.

**Purpose**:
- Codifies infrastructure as version-controlled code
- Enables reproducible deployments (destroy and recreate anytime)
- Supports both local development (Docker Compose) and AWS production (Terraform)
- Implements auto-scaling based on CloudWatch metrics

**AWS Resources Provisioned (47 total)**:
- **Networking**: VPC, 2 public subnets, 2 private subnets, Internet Gateway, NAT Gateway, route tables
- **Compute**: EC2 Auto Scaling Group (2-10 instances), Launch Template, Application Load Balancer, Target Group
- **Message Queue**: MSK Kafka cluster (2 brokers, kafka.t3.small, 10 partitions)
- **Database**: RDS PostgreSQL (db.t3.micro, 20GB storage, automated backups)
- **Cache**: ElastiCache Redis (cache.t3.micro, single node)
- **Security**: 6 Security Groups (Publisher, Forge, ALB, Kafka, RDS, Redis)
- **IAM**: EC2 instance roles, CloudWatch policies, MSK access policies
- **Monitoring**: CloudWatch Dashboard, 4 alarms (high lag, low lag, high CPU), auto-scaling policies

**Terraform Structure**:
```
terraform/
├── main.tf              # Provider configuration, AWS region
├── variables.tf         # Configurable parameters (instance types, counts, regions)
├── outputs.tf           # Endpoint URLs, IPs, resource IDs
├── vpc.tf              # VPC, subnets, route tables, NAT Gateway
├── security-groups.tf  # Security groups for all services
├── msk.tf              # MSK Kafka cluster configuration
├── rds.tf              # RDS PostgreSQL instance
├── redis.tf            # ElastiCache Redis cluster
├── ec2.tf              # EC2 instances, Auto Scaling Group, Load Balancer
├── iam.tf              # IAM roles for EC2, CloudWatch, MSK access
├── cloudwatch.tf       # Alarms, auto-scaling policies, dashboard
└── user-data/
    ├── trade-publisher.sh  # Bootstrap script for Publisher EC2
    └── trade-forge.sh      # Bootstrap script for Forge workers
```

**Docker Compose (Local Development)**:
```
docker-compose.yml
├── Kafka (confluent/cp-kafka:7.5.0) - 1 broker, 10 partitions
├── PostgreSQL (postgres:15-alpine) - trade_recon database
├── Redis (redis:7-alpine) - job tracking cache
├── Trade-Publisher (built from ../trade-publisher)
├── Trade-Forge × 3 instances (built from ../trade-forge)
├── Prometheus (prom/prometheus) - metrics collection
└── Grafana (grafana/grafana) - monitoring dashboards
```

**Grafana Dashboard Panels**:
- Trades Processed per Second (by type & instance)
- Total Trades Processed (cumulative counter)
- Processing Latency (average in milliseconds)
- Validation Failures per Second
- Kafka Consumer Lag (messages behind)
- Active Processing Count (real-time gauge)
- Trade Distribution by Type (pie chart)
- Load Distribution Across Instances (pie chart)

**Auto-Scaling Configuration**:
- **Minimum**: 2 instances
- **Maximum**: 10 instances
- **Desired**: 3 instances
- **Scale-up trigger**: Kafka consumer lag > 1000 messages OR CPU > 70%
- **Scale-down trigger**: Kafka consumer lag < 100 messages for 5 minutes
- **Scale-up action**: Add 2 instances, 5-minute cooldown
- **Scale-down action**: Remove 1 instance, 10-minute cooldown

**User-Data Bootstrap Process**:
1. Update system packages (yum update)
2. Install Docker and Docker Compose
3. Install Git and clone application repository from GitHub
4. Create environment-specific configuration (application-aws.yml)
5. Build Docker image from Dockerfile
6. Run container with environment variables (Kafka, RDS, Redis endpoints)
7. Install and configure CloudWatch agent
8. Start CloudWatch agent for system metrics

**CloudWatch Metrics Published**:
- `TradeReconciliation/KafkaConsumerLag` - Messages behind per instance
- `AWS/EC2/CPUUtilization` - CPU usage per Auto Scaling Group
- `AWS/AutoScaling/GroupDesiredCapacity` - Target instance count
- `AWS/AutoScaling/GroupInServiceInstances` - Active healthy instances

**Network Architecture**:
```
Internet
    ↓
Internet Gateway
    ↓
Public Subnets (us-east-1a, us-east-1b)
    ├── Trade-Publisher EC2 (public IP)
    ├── Application Load Balancer
    └── NAT Gateway
         ↓
Private Subnets (us-east-1a, us-east-1b)
    ├── Trade-Forge Auto Scaling Group (2-10 instances)
    ├── MSK Kafka Cluster (2 brokers)
    ├── RDS PostgreSQL
    └── ElastiCache Redis
```

**Security Groups**:
- **Trade-Publisher**: SSH (22), HTTP API (8089) from anywhere
- **Trade-Forge**: SSH (22), Prometheus (8090) from VPC, ALB health checks
- **ALB**: HTTP (80) from anywhere
- **Kafka**: Broker (9092), Zookeeper (2181) from VPC only
- **RDS**: PostgreSQL (5432) from VPC only
- **Redis**: Redis (6379) from VPC only

**Deployment Commands**:
```bash
# Initialize Terraform
terraform init

# Preview changes
terraform plan

# Deploy infrastructure
terraform apply

# Destroy all resources
terraform destroy

# Local development
docker-compose up --build -d

# View logs
docker-compose logs -f trade-forge-1
```

**Configuration Variables**:
- `aws_region`: us-east-1 (N. Virginia)
- `vpc_cidr`: 10.0.0.0/16
- `kafka_instance_type`: kafka.t3.small
- `ec2_instance_type`: t3.medium
- `trade_forge_min_size`: 2
- `trade_forge_max_size`: 10
- `db_username`: dbadmin
- `kafka_version`: 3.6.0

**Cost Estimation (48 hours)**:
- MSK Kafka (2 × kafka.t3.small): ~$4
- RDS PostgreSQL (db.t3.micro): ~$1
- ElastiCache Redis (cache.t3.micro): ~$0.80
- EC2 instances (3 × t3.medium): ~$6
- NAT Gateway, Load Balancer, data transfer: ~$1
- **Total**: ~$12-15 for 48-hour test deployment

**Terraform Outputs**:
- `trade_publisher_public_ip` - HTTP API endpoint
- `load_balancer_dns` - ALB DNS for Trade-Forge cluster
- `kafka_bootstrap_brokers` - Kafka connection string
- `rds_endpoint` - PostgreSQL connection endpoint
- `redis_endpoint` - Redis connection endpoint
- `trade_forge_asg_name` - Auto Scaling Group identifier

**Technology**: Terraform 1.0+, AWS Provider 5.0, Docker, Docker Compose, Bash, CloudWatch, Prometheus, Grafana

---
### 6. Infrastructure Components

#### Kafka
- **Purpose**: Message queue buffering trades between services
- **Configuration**: 
  - Topic: `trade-recon-input`
  - Partitions: 10 (for parallel consumption)
  - Replication: 1 (local), 3 (production)
- **Image**: `confluentinc/cp-kafka:latest`

#### PostgreSQL
- **Purpose**: Shadow database for trade booking simulation
- **Configuration**:
  - Database: `trade_recon`
  - Schema: `trades` table with full trade fields
  - Transactions: Always rolled back (no persistence)
- **Image**: `postgres:15-alpine`

#### Redis
- **Purpose**: Job tracking for async reconciliation runs
- **Data Structure**: 
```
  Key: job:{jobId}
  Value: { status, progress, startTime, endTime }
  TTL: 24 hours
```
- **Image**: `redis:7-alpine`

#### Prometheus
- **Purpose**: Metrics collection from all services
- **Metrics Tracked**:
  - Trades generated/published (counter)
  - Trades processed (counter)
  - Processing latency (histogram)
  - Kafka consumer lag (gauge)
  - Job duration (histogram)
- **Scrape Interval**: 15 seconds

#### Grafana
- **Purpose**: Visualization dashboards
- **Dashboards**:
  - System Overview (all services)
  - Data Factory Metrics (generation throughput)
  - Sherpa Worker Metrics (processing latency, error rates)
  - Kafka Metrics (lag, partition distribution)

---

## Quick Start

### Prerequisites
- Docker & Docker Compose
- Java 17+ (for local development)
- Python 3.11+ (for Sentinel development)
- Maven (for building Java services)

### Local Development

1. **Clone the repository**:
```bash
git clone https://github.com/topnotchtrash/trade-reconciliation-system.git
cd trade-reconciliation-system
```

2. **Start infrastructure**:
```bash
docker-compose up -d kafka postgres redis prometheus grafana
```

3. **Build and run Data Factory**:
```bash
cd data-factory
mvn clean package
docker build -t data-factory:latest .
docker run -p 8080:8080 --network host data-factory:latest
```

4. **Build and run Sherpa Workers** (in 3 terminals):
```bash
cd sherpa-worker
mvn clean package
docker build -t sherpa-worker:latest .

# Terminal 1
docker run --network host sherpa-worker:latest

# Terminal 2  
docker run --network host sherpa-worker:latest

# Terminal 3
docker run --network host sherpa-worker:latest
```

5. **Build and run Sentinel**:
```bash
cd sentinel
pip install -r requirements.txt
docker build -t sentinel:latest .
docker run -p 8000:8000 --network host sentinel:latest
```

6. **Trigger a reconciliation job**:
```bash
curl -X POST http://localhost:8080/api/reconciliation/start \
  -H "Content-Type: application/json" \
  -d '{
    "tradeCount": 10000,
    "tradeDistribution": {
      "INTEREST_RATE_SWAP": 30,
      "EQUITY_SWAP": 30,
      "FX_FORWARD": 20,
      "EQUITY_OPTION": 15,
      "CREDIT_DEFAULT_SWAP": 5
    },
    "profileDistribution": {
      "CLEAN": 70,
      "EDGE_CASE": 20,
      "STRESS": 10
    }
  }'
```

7. **Monitor progress**:
- Grafana: http://localhost:3000 (admin/admin)
- Prometheus: http://localhost:9090
- Job Status: `curl http://localhost:8080/api/reconciliation/status/{jobId}`

8. **View reconciliation report**:
```bash
curl http://localhost:8000/api/reconciliation/report/{jobId}
```

---

## Docker Compose (All-in-One)

For the easiest setup, use the provided `docker-compose.yml`:
```bash
# Start everything
docker-compose up -d

# View logs
docker-compose logs -f

# Stop everything
docker-compose down
```

This starts:
- Zookeeper + Kafka
- PostgreSQL
- Redis
- Data Factory (1 instance)
- Sherpa Workers (3 instances)
- Sentinel
- Prometheus
- Grafana

---

## Configuration

### Environment Variables

**Data Factory**:
```bash
KAFKA_BOOTSTRAP_SERVERS=localhost:9092
REDIS_HOST=localhost
REDIS_PORT=6379
BATCH_SIZE=100
THREAD_POOL_SIZE=8
```

**Sherpa Workers**:
```bash
KAFKA_BOOTSTRAP_SERVERS=localhost:9092
POSTGRES_URL=jdbc:postgresql://localhost:5432/trade_recon
POSTGRES_USER=admin
POSTGRES_PASSWORD=password
EXECUTION_MODE=ROLLBACK
```

**Sentinel**:
```bash
DATABASE_URL=postgresql://admin:password@localhost:5432/trade_recon
```

### Profiles

**Local Development** (`application-local.yml`):
- Kafka: localhost:9092
- PostgreSQL: localhost:5432
- Redis: localhost:6379
- Log level: DEBUG

**AWS Deployment** (`application-aws.yml`):
- Kafka: EC2 private IP
- PostgreSQL: RDS endpoint
- Redis: EC2 private IP
- Log level: INFO
---
## Monitoring & Observability

### Prometheus Metrics

**Data Factory**:
- `trades_generated_total` - Total trades generated
- `trades_published_total` - Total trades published to Kafka
- `kafka_publish_latency_ms` - Kafka publish latency
- `job_duration_seconds` - Job completion time

**Sherpa Workers**:
- `trades_processed_total` - Total trades processed
- `trades_failed_total` - Total processing failures
- `processing_latency_ms` - Trade processing latency
- `db_rollback_total` - Total transaction rollbacks

**Kafka**:
- `kafka_consumer_lag` - Consumer group lag
- `kafka_partition_offset` - Current partition offsets

### Grafana Dashboards

1. **System Overview**
   - Trades/sec across all services
   - End-to-end latency
   - Error rates
   - Active jobs

2. **Data Factory Dashboard**
   - Generation throughput
   - Kafka publish rate
   - Job queue depth
   - Memory usage

3. **Sherpa Workers Dashboard**
   - Processing latency (p50, p95, p99)
   - Success/failure rates by trade type
   - Database connection pool
   - Consumer lag

4. **Kafka Dashboard**
   - Topic throughput
   - Partition distribution
   - Consumer group lag
   - Broker health

---

## Testing

### Unit Tests
```bash
# Data Factory
cd data-factory
mvn test

# Sherpa Workers
cd sherpa-worker
mvn test

# Sentinel
cd sentinel
pytest
```

### Integration Tests
```bash
# End-to-end test
./scripts/integration-test.sh
```

**Test Flow**:
1. Start all services via Docker Compose
2. Trigger 1K trade reconciliation
3. Verify Kafka messages published
4. Verify trades processed in PostgreSQL (then rolled back)
5. Verify reconciliation report matches expected
6. Teardown

### Load Testing
```bash
# 100K trades
curl -X POST http://localhost:8089/api/reconciliation/start \
  -d '{"tradeCount": 100000, ...}'
```

---

---

## Roadmap

### Completed 
- [x] Annapurna library (published to Maven Central)
- [x] Data Factory service
- [x] Sherpa worker service
- [x] Sentinel reconciliation service
- [x] Docker Compose local setup
- [x] Prometheus metrics integration
- [x] Grafana dashboards

### In Progress 
- [ ] AWS deployment with Terraform
- [ ] Auto-scaling workers based on Kafka lag
- [ ] Advanced reconciliation reports (PDF export)

### Future Enhancements 
- [ ] Kubernetes deployment manifests
- [ ] CI/CD pipeline (GitHub Actions)
- [ ] Performance benchmarking suite
- [ ] Historical metrics storage (TimescaleDB)
- [ ] Web UI for job management
- [ ] Slack/email notifications for job completion

---

## Contributing

This is a personal portfolio project, but feedback and suggestions are welcome!

1. Fork the repository
2. Create a feature branch (`git checkout -b feature/improvement`)
3. Commit changes (`git commit -am 'Add improvement'`)
4. Push to branch (`git push origin feature/improvement`)
5. Open a Pull Request

---

## License

MIT License - see [LICENSE](LICENSE) file for details

---

## Author

**Your Name**  
- GitHub: [@topnotchtrash](https://github.com/topnotchtrash)
- LinkedIn: [Your LinkedIn](https://linkedin.com/in/yourprofile)
- Email: your.email@example.com

---

## Acknowledgments

- Inspired by real-world trade reconciliation systems at financial institutions
- Built to demonstrate distributed systems, event-driven architecture, and financial domain knowledge
- Special thanks to the Spring Boot, Kafka, and FastAPI communities

---

## FAQ

**Q: Can this be used in production?**  
A: This is a demonstration/learning project. While it uses production-grade patterns, it would need hardening (HA, security, monitoring) for actual production use.

**Q: Why not use managed services (AWS MSK, ElastiCache)?**  
A: Cost optimization for local development. Self-hosted Kafka/Redis keeps costs near zero locally and <$30 for AWS deployment.

**Q: Can I add more trade types?**  
A: Yes! Extend Annapurna library with new generators, then add corresponding processors in Sherpa.

**Q: How does transaction rollback work?**  
A: Each trade is processed inside a database transaction. After executing the INSERT, we call `transaction.setRollbackOnly()` to discard changes.

**Q: What if Kafka goes down mid-processing?**  
A: Workers will retry based on Kafka consumer configuration. Unprocessed messages remain in Kafka until workers recover.

**Q: Can I deploy this to Kubernetes?**  
A: Yes! The Docker images can run in K8s with appropriate manifests. KEDA can auto-scale workers based on Kafka lag.

---

**Last Updated**: December 2024