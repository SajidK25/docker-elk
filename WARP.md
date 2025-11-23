# WARP.md

This file provides guidance to WARP (warp.dev) when working with code in this repository.

## Overview

This is a Docker-based deployment of the Elastic stack (ELK) - Elasticsearch, Logstash, and Kibana. It's designed as a **development template**, not a production-ready blueprint. The stack uses Docker Compose to orchestrate services and emphasizes minimal, unopinionated configuration that promotes exploration.

**Current Elastic Stack version:** 9.2.1

## Essential Commands

### Initial Setup
```bash
# Initialize Elasticsearch users and groups (run once)
docker compose up setup

# Generate Kibana encryption keys (highly recommended)
docker compose up kibana-genkeys
# Copy the output to kibana/config/kibana.yml
```

### Starting/Stopping the Stack
```bash
# Start all services
docker compose up

# Start in detached mode
docker compose up -d

# Stop services
docker compose down

# Complete teardown with data removal
docker compose --profile=setup down -v
```

### Building and Rebuilding
```bash
# Rebuild all images (required after changing ELASTIC_VERSION or switching branches)
docker compose build

# Rebuild specific service
docker compose build elasticsearch
docker compose build logstash
docker compose build kibana
```

### Monitoring and Logs
```bash
# View logs for all services
docker compose logs

# Follow logs for specific service
docker compose logs -f elasticsearch
docker compose logs -f logstash
docker compose logs -f kibana

# Check service status
docker compose ps
```

### User and Password Management
```bash
# Reset password for elastic user
docker compose exec elasticsearch bin/elasticsearch-reset-password --batch --user elastic

# Reset password for logstash_internal user
docker compose exec elasticsearch bin/elasticsearch-reset-password --batch --user logstash_internal

# Reset password for kibana_system user
docker compose exec elasticsearch bin/elasticsearch-reset-password --batch --user kibana_system

# After resetting passwords, update .env file and restart services:
docker compose up -d logstash kibana
```

### Testing Data Ingestion
```bash
# Inject log data via TCP (port 50000) using netcat
cat /path/to/logfile.log | nc -q0 localhost 50000    # BSD
cat /path/to/logfile.log | nc -c localhost 50000     # GNU
cat /path/to/logfile.log | nc --send-only localhost 50000  # nmap
```

## Architecture

### Service Components

**Elasticsearch** (Port 9200, 9300)
- Core search and analytics engine
- Single-node configuration by default (discovery.type: single-node)
- Data persisted in Docker volume
- Security enabled with trial license (30 days)
- Configuration: `elasticsearch/config/elasticsearch.yml`
- JVM heap: 512MB (set via ES_JAVA_OPTS)

**Logstash** (Ports 5044, 50000, 9600)
- Data processing pipeline
- Accepts input from Beats (port 5044) and TCP (port 50000)
- Outputs to Elasticsearch using logstash_internal user
- Configuration: `logstash/config/logstash.yml`
- Pipeline: `logstash/pipeline/logstash.conf`
- JVM heap: 256MB (set via LS_JAVA_OPTS)

**Kibana** (Port 5601)
- Web UI for Elasticsearch data visualization
- Default credentials: elastic/changeme (change in production)
- Connects to Elasticsearch using kibana_system user
- Configuration: `kibana/config/kibana.yml`
- Includes Fleet and APM server configuration

**Setup Service** (setup profile)
- One-time initialization service
- Creates users: elastic, logstash_internal, kibana_system
- Creates custom roles as defined in `setup/roles/`
- Passwords defined in `.env` file
- Uses `setup/entrypoint.sh` and `setup/lib.sh`

### Configuration Flow

1. Environment variables in `.env` define passwords and ELASTIC_VERSION
2. docker-compose.yml uses these to configure service environments
3. Service-specific configs (elasticsearch.yml, logstash.yml, kibana.yml) reference environment variables
4. Setup service initializes users with passwords from `.env`

### Key Files

- `.env` - All passwords and version configuration
- `docker-compose.yml` - Service orchestration and network topology
- `elasticsearch/Dockerfile`, `logstash/Dockerfile`, `kibana/Dockerfile` - Custom image definitions
- `setup/entrypoint.sh` - User initialization script
- `logstash/pipeline/logstash.conf` - Log processing pipeline definition

### Extensions System

The `extensions/` directory contains optional integrations:
- curator - Index management
- filebeat - Log file shipping
- fleet - Elastic Agent management
- heartbeat - Uptime monitoring
- metricbeat - Metrics collection

Extensions require manual configuration and are not enabled by default. Refer to individual extension README files.

### Networking

All services run on the `elk` bridge network. Services communicate using Docker service names:
- Elasticsearch: `elasticsearch:9200`
- Logstash: `logstash:5044`, `logstash:50000`
- Kibana: `kibana:5601`

## Version Management

To change Elastic Stack version:
1. Update `ELASTIC_VERSION` in `.env`
2. Rebuild all images: `docker compose build`
3. Review official upgrade docs before upgrading existing deployments

## Security Notes

- Bootstrap checks are disabled for development convenience
- Default passwords are "changeme" - **change immediately after setup**
- Trial license enables Platinum features for 30 days
- After trial, stack reverts to Basic license automatically
- To disable paid features before trial: use License Management UI or Licensing API

## Common Modifications

### Adjust Memory Allocation
Edit `docker-compose.yml` environment sections:
```yaml
ES_JAVA_OPTS: -Xms1g -Xmx1g  # For Elasticsearch
LS_JAVA_OPTS: -Xms512m -Xmx512m  # For Logstash
```

### Add Logstash Plugins
1. Add RUN statement to `logstash/Dockerfile`
2. Update `logstash/pipeline/logstash.conf` with plugin config
3. Rebuild: `docker compose build logstash`

### Override Service Configuration
Add environment variables to service definitions in `docker-compose.yml`:
```yaml
elasticsearch:
  environment:
    cluster.name: my-cluster
    network.host: _non_loopback_
```

## Access Points

- Kibana UI: http://localhost:5601
- Elasticsearch HTTP API: http://localhost:9200
- Logstash monitoring API: http://localhost:9600
- Logstash Beats input: localhost:5044
- Logstash TCP input: localhost:50000
