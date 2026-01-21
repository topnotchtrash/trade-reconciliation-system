# Trade Reconciliation System

> Event-driven financial pipeline simulator for validating trade processing at scale using synthetic data
[![Maven Central](https://img.shields.io/maven-central/v/io.github.topnotchtrash/annapurna.svg)](https://central.sonatype.com/artifact/io.github.topnotchtrash/annapurna)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)


Note- Still Under Development( The final updates will be posted on 02/28/2026)
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

### 1. Annapurna (Synthetic Data Generator)

**Repository**: [github.com/topnotchtrash/annapurna](https://github.com/topnotchtrash/annapurna)  
**Maven Central**: [io.github.topnotchtrash:annapurna](https://central.sonatype.com/artifact/io.github.topnotchtrash/annapurna)

Standalone Java library for generating realistic synthetic financial trade data.

**Features**:
- Generates 5 trade types: Interest Rate Swaps, Equity Swaps, FX Forwards, Equity Options, Credit Default Swaps
- 3 data quality profiles: CLEAN (70%), EDGE_CASE (20%), STRESS (10%)
- Multi-threaded generation (1M+ trades/second)
- Configurable distributions and date ranges
- Zero PII/sensitive data

**Usage**:
```java
List<Trade> trades = Annapurna.builder()
    .tradeTypes()
        .interestRateSwap(30)
        .equitySwap(30)
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

**Technology**: Java 17+, Maven, JavaFaker, Jackson

---

### 2. Data Factory (Orchestration Service)

**Repository**: [github.com/topnotchtrash/trade-publisher](https://github.com/topnotchtrash/trade-publisher)

Spring Boot microservice that orchestrates trade generation and publishes to Kafka.

**Responsibilities**:
- REST API for triggering reconciliation jobs
- Integrates Annapurna library for trade generation
- Publishes trades to Kafka in batches (100 trades/batch)
- Tracks job progress in Redis
- Exposes metrics for Prometheus

**API Endpoints**:
```bash
# Start reconciliation job
POST /api/reconciliation/start
{
  "tradeCount": 100000,
  "tradeDistribution": { "EQUITY_SWAP": 40, ... },
  "profileDistribution": { "CLEAN": 70, ... }
}

# Get job status
GET /api/reconciliation/status/{jobId}
```

**Technology**: Spring Boot, Spring Kafka, Spring Data Redis, Maven, Docker

**Configuration**:
- Kafka topic: `trade-recon-input`
- Redis TTL: 24 hours
- Batch size: 100 trades
- Thread pool: 8 threads

---

### 3. Sherpa (Worker Service)

**Repository**: [github.com/topnotchtrash/sherpa](https://github.com/topnotchtrash/sherpa-worker)

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

### 4. Sentinel (Reconciliation Service)

**Repository**: [github.com/topnotchtrash/sentinel-reconciliation](https://github.com/topnotchtrash/sentinel-reconciliation)

Python FastAPI service that validates end-to-end processing results.

**Responsibilities**:
- Queries shadow database for processed trades
- Compares expected vs actual results
- Identifies discrepancies (missing trades, failed validations, data mismatches)
- Generates reconciliation reports
- Exposes results via REST API

**Reconciliation Logic**:
```python
# Compare input vs output
expected_trades = get_kafka_message_count()
processed_trades = query_shadow_db()

discrepancies = {
    "missing": expected - processed,
    "failed_validation": count_failed_trades(),
    "data_mismatches": compare_field_values()
}
```

**API Endpoints**:
```bash
# Get reconciliation report
GET /api/reconciliation/report/{jobId}

# Get discrepancy details
GET /api/reconciliation/discrepancies/{jobId}
```

**Technology**: Python 3.11, FastAPI, SQLAlchemy, Pandas, PostgreSQL, Docker

**Reports Generated**:
- Trade count summary (expected vs processed)
- Success/failure breakdown by trade type
- Processing time percentiles (p50, p95, p99)
- Error analysis (top failure reasons)

---

### 5. Infrastructure Components

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