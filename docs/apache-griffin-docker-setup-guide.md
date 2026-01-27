# Apache Griffin Docker Setup Guide

This guide provides detailed instructions for setting up Apache Griffin using Docker.

## Docker Compose Files Analysis

### 1. Batch Mode (docker-compose-batch.yml)

**Services: 2 containers**

| Service | Image | Purpose |
|---------|-------|---------|
| `griffin` | `apachegriffin/griffin_spark2:0.3.0` | Main Griffin stack |
| `es` | `apachegriffin/elasticsearch` | Metrics storage |

**Port Mappings (griffin container):**

| Host Port | Container Port | Service |
|-----------|----------------|---------|
| 38080 | 8080 | **Griffin UI** |
| 38088 | 8088 | YARN ResourceManager UI |
| 38042 | 8042 | YARN NodeManager UI |
| 38998 | 8998 | Livy (Spark REST) |
| 39083 | 9083 | Hive Metastore |
| 33306 | 3306 | MySQL |
| 35432 | 5432 | PostgreSQL |
| 32122 | 2122 | SSH |

**Port Mappings (es container):**

| Host Port | Container Port | Service |
|-----------|----------------|---------|
| 39200 | 9200 | Elasticsearch HTTP |
| 39300 | 9300 | Elasticsearch Transport |

---

### 2. Streaming Mode (docker-compose-streaming.yml)

**Services: 4 containers** (adds Zookeeper + Kafka)

| Service | Image | Extra Ports |
|---------|-------|-------------|
| `griffin` | `apachegriffin/griffin_spark2:0.3.0` | Same as batch |
| `es` | `apachegriffin/elasticsearch` | 39200, 39300 |
| `zk` | `zookeeper:3.5` | 32181 -> 2181 |
| `kafka` | `apachegriffin/kafka` | 39092 -> 9092 |

**Additional environment variables for griffin container:**
- `ZK_HOSTNAME: zk`
- `KAFKA_HOSTNAME: kafka`

---

## Setup Steps

### Prerequisites

**Step 1: Install Docker & Docker Compose**
```bash
# Follow official docs:
# https://docs.docker.com/engine/installation/
# https://docs.docker.com/compose/install/
```

**Step 2: Configure system for Elasticsearch**
```bash
# Linux
sudo sysctl -w vm.max_map_count=262144

# macOS: Increase Docker memory to 4GB+ in Docker Desktop preferences
```

**Step 3: Pull Docker images**
```bash
docker pull apachegriffin/griffin_spark2:0.3.0
docker pull apachegriffin/elasticsearch
docker pull apachegriffin/kafka
docker pull zookeeper:3.5
```

---

### Batch Mode Setup

```bash
# 1. Navigate to compose directory
cd griffin-doc/docker/compose

# 2. Start containers (wait ~1 minute for services to initialize)
docker-compose -f docker-compose-batch.yml up -d

# 3. Verify containers are running
docker container ls
```

**After startup, access:**
- Griffin UI: `http://localhost:38080`
- YARN UI: `http://localhost:38088`
- Elasticsearch: `http://localhost:39200`

**Test the API:**
```bash
# Check Griffin version
curl http://localhost:38080/api/v1/version
```

**Using Postman:**
1. Import config files from `griffin-doc/service/postman/`
2. Set `BASE_PATH` environment variable to `<your-ip>:38080`
3. Test with `Basic -> Get griffin version`

**Create a measure and job:**
1. `Measures -> Add measure` (creates accuracy measure)
2. `Jobs -> Add job` (schedules execution every 5 mins)

**Query metrics from Elasticsearch:**
```bash
curl -XGET 'localhost:39200/griffin/accuracy/_search?pretty&filter_path=hits.hits._source' \
  -d '{"query":{"match_all":{}}, "sort": [{"tmst": {"order": "asc"}}]}'
```

---

### Streaming Mode Setup

```bash
# 1. Start all 4 containers
cd griffin-doc/docker/compose
docker-compose -f docker-compose-streaming.yml up -d

# 2. Enter the griffin container
docker exec -it griffin bash

# 3. Navigate to measure directory
cd ~/measure

# 4. Run streaming accuracy measurement
./streaming-accu.sh

# 5. Monitor logs
tail -f streaming-accu.log
```

**To switch to profiling mode:**
```bash
# Kill current streaming process
kill -9 `ps -ef | awk '/griffin-measure/{print $2}'`

# Clear checkpoint directories
./clear.sh

# Run streaming profiling
./streaming-prof.sh
tail -f streaming-prof.log
```

---

## Summary

| Mode | Containers | Use Case |
|------|------------|----------|
| **Batch** | 2 (griffin + es) | Scheduled data quality checks |
| **Streaming** | 4 (+ zk + kafka) | Real-time data quality monitoring |

The Docker setup provides a complete single-node environment with Hadoop, Hive, Spark, and all Griffin components pre-configured with demo data.

---

## References

- Original Docker Guide: `griffin-doc/docker/griffin-docker-guide.md`
- Batch Compose File: `griffin-doc/docker/compose/docker-compose-batch.yml`
- Streaming Compose File: `griffin-doc/docker/compose/docker-compose-streaming.yml`
- Postman Collections: `griffin-doc/service/postman/`
