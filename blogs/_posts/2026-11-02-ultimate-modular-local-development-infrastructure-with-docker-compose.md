---
layout: blog
title: "The Ultimate Modular Local Development Infrastructure with Docker Compose"
category: Tech
---

Modern software development requires a lot of moving parts. A typical application today might rely on a primary database, a caching layer, a search engine, monitoring tools, and increasingly, local AI models. Managing all these dependencies locally can be a nightmare of conflicting versions, complex setup scripts, and the dreaded "it works on my machine" syndrome. 

<br>

I recently ran into this exact problem at my current job. I was working on a project that required heavily experimenting with several different types of databases—document stores, vector databases, search engines, and caching layers—to find the right fit for a new architecture. Setting them up natively one by one was tedious, and keeping their configurations in sync was becoming a headache. I needed a way to spin up (and tear down) complex database combinations instantly.

<br>

In this post, we'll explore the robust, production-like local development infrastructure I built using a single `docker-compose.yml` file to solve this problem. This setup isn't just a monolithic block of services; it's a highly modular, scalable, and resilient environment designed to maximize developer productivity and experimentation.

<br>

You can find the complete code for this setup in my GitHub repository: **[local-dev-infrastructure](https://github.com/itsujji/local-dev-infrastructure)**.

<br>

Here is a look at the architecture we are building:

```yaml
# docker-compose.yml
name: local-dev-infrastructure

services:
  # ==========================================
  # CORE INFRASTRUCTURE
  # Starts by default
  # ==========================================

  mongodb:
    image: mongo:8
    container_name: senserank-mongodb
    restart: unless-stopped
    environment:
      MONGO_INITDB_ROOT_USERNAME: ${MONGO_USERNAME:-admin}
      MONGO_INITDB_ROOT_PASSWORD: ${MONGO_PASSWORD:-password}
    ports:
      - "27017:27017"
    volumes:
      - mongodb-data:/data/db
    networks:
      - local-infra-network
    healthcheck:
      test:
        - CMD
        - mongosh
        - --quiet
        - --username
        - ${MONGO_USERNAME:-admin}
        - --password
        - ${MONGO_PASSWORD:-password}
        - --authenticationDatabase
        - admin
        - --eval
        - "db.adminCommand('ping').ok"
      interval: 15s
      timeout: 5s
      retries: 10
      start_period: 20s

  opensearch:
    image: opensearchproject/opensearch:2.19.1
    container_name: senserank-opensearch
    restart: unless-stopped
    environment:
      discovery.type: single-node
      DISABLE_SECURITY_PLUGIN: "true"
      OPENSEARCH_JAVA_OPTS: "-Xms512m -Xmx512m"
    ulimits:
      memlock:
        soft: -1
        hard: -1
    ports:
      - "9200:9200"
    volumes:
      - opensearch-data:/usr/share/opensearch/data
    networks:
      - local-infra-network
    healthcheck:
      test:
        - CMD-SHELL
        - curl -fsS http://localhost:9200 >/dev/null || exit 1
      interval: 20s
      timeout: 10s
      retries: 20
      start_period: 40s

  redis:
    image: redis:7-alpine
    container_name: senserank-redis
    restart: unless-stopped
    ports:
      - "6379:6379"
    volumes:
      - redis-data:/data
    networks:
      - local-infra-network
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 10

  documentdb:
    image: ghcr.io/documentdb/documentdb/documentdb-local:latest
    container_name: senserank-docdb
    restart: unless-stopped
    command:
      - "--username"
      - ${DOCDB_USERNAME:-admin}
      - "--password"
      - ${DOCDB_PASSWORD:-password}
    ports:
      - "10260:10260"
    volumes:
      - docdb-data:/data
    networks:
      - local-infra-network
    healthcheck:
      test:
        - CMD
        - mongosh
        - localhost:10260
        - -u
        - ${DOCDB_USERNAME:-admin}
        - -p
        - ${DOCDB_PASSWORD:-password}
        - --authenticationMechanism
        - SCRAM-SHA-256
        - --tls
        - --tlsAllowInvalidCertificates
        - --eval
        - "db.adminCommand('ping').ok"
      interval: 15s
      timeout: 5s
      retries: 10
      start_period: 20s

  # ==========================================
  # AI / ML PROFILE
  # docker compose --profile ai up -d
  # ==========================================

  ollama:
    image: ollama/ollama:latest
    container_name: senserank-ollama
    restart: unless-stopped
    profiles:
      - ai
    ports:
      - "11434:11434"
    volumes:
      - ollama-data:/root/.ollama
    networks:
      - local-infra-network

  # ==========================================
  # DATABASE PROFILE
  # docker compose --profile database up -d
  # ==========================================

  postgresql:
    image: pgvector/pgvector:pg17
    container_name: senserank-postgresql
    restart: unless-stopped
    profiles:
      - database
    environment:
      POSTGRES_USER: ${POSTGRES_USER:-admin}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:-password}
      POSTGRES_DB: ${POSTGRES_DB:-senserank}
    ports:
      - "5432:5432"
    volumes:
      - postgresql-data:/var/lib/postgresql/data
    networks:
      - local-infra-network
    healthcheck:
      test:
        - CMD-SHELL
        - pg_isready -U "$$POSTGRES_USER" -d "$$POSTGRES_DB"
      interval: 10s
      timeout: 5s
      retries: 10
      start_period: 10s

  # ==========================================
  # MONITORING PROFILE
  # docker compose --profile monitoring up -d
  # ==========================================

  prometheus:
    image: prom/prometheus:latest
    container_name: senserank-prometheus
    restart: unless-stopped
    profiles:
      - monitoring
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - prometheus-data:/prometheus
    command:
      - "--config.file=/etc/prometheus/prometheus.yml"
      - "--storage.tsdb.path=/prometheus"
      - "--storage.tsdb.retention.time=15d"
    networks:
      - local-infra-network
    healthcheck:
      test:
        - CMD
        - wget
        - --spider
        - -q
        - http://localhost:9090/-/healthy
      interval: 15s
      timeout: 5s
      retries: 10

  grafana:
    image: grafana/grafana:latest
    container_name: senserank-grafana
    restart: unless-stopped
    profiles:
      - monitoring
    environment:
      GF_SECURITY_ADMIN_USER: ${GRAFANA_USER:-admin}
      GF_SECURITY_ADMIN_PASSWORD: ${GRAFANA_PASSWORD:-admin}
      GF_USERS_ALLOW_SIGN_UP: "false"
    ports:
      - "3000:3000"
    volumes:
      - grafana-data:/var/lib/grafana
    depends_on:
      prometheus:
        condition: service_healthy
    networks:
      - local-infra-network

# ==========================================
# NETWORKS
# ==========================================

networks:
  local-infra-network:
    name: local-infra-network
    driver: bridge

# ==========================================
# PERSISTENT VOLUMES
# ==========================================

volumes:
  mongodb-data:
  opensearch-data:
  redis-data:
  docdb-data:
  ollama-data:
  postgresql-data:
  prometheus-data:
  grafana-data:
```

<br>

Let's dive into why this specific setup could be a game-changer for engineering teams.

<br>

## 1. All-in-One Local Infrastructure
Instead of installing MongoDB, Redis, OpenSearch, and PostgreSQL natively on your operating system, this setup containerizes everything. You get a complete, enterprise-grade backend stack running on your machine in seconds. 

<br>

*   **MongoDB & DocumentDB:** For robust NoSQL data storage (including AWS DocumentDB emulation for seamless cloud parity).
*   **Redis:** For high-performance caching and message brokering.
*   **OpenSearch:** For powerful full-text search and log analytics.
*   **PostgreSQL (with pgvector):** For relational data and advanced vector similarity search.


<br>

## 2. No Cloud Dependency for Basic Development
Cloud development environments are great, but relying on them for every minor code change can slow you down, incur unnecessary costs, and require a constant internet connection. 

<br>

This Docker Compose setup entirely removes the cloud dependency for basic development. You can code on an airplane, a train, or in a coffee shop with spotty Wi-Fi. Because it includes an AWS DocumentDB local emulator and local AI capabilities via Ollama, you can build and test complex, cloud-native, AI-driven applications entirely offline.

<br>

## 3. The Magic of Docker Compose Profiles (Modular Infrastructure)
This is where the setup truly shines. Running 8 heavy containers simultaneously will drain the battery and hog the RAM of even the most powerful developer laptops. 

<br>

By utilizing **Docker Compose Profiles**, this infrastructure is entirely modular. Services are grouped logically, meaning you only spin up what you need for your current task.

<br>

*   **Core Services:** Run `docker compose up -d`. This starts the essentials: MongoDB, OpenSearch, Redis, and DocumentDB.
*   **Working on a new LLM feature?** Run `docker compose --profile ai up -d`. This spins up the core services *plus* Ollama, allowing you to run local AI models without touching external APIs.
*   **Need relational data and vector search?** Run `docker compose --profile database up -d` to bring PostgreSQL into the mix.
*   **Debugging performance?** Run `docker compose --profile monitoring up -d` to attach Prometheus and Grafana to your local stack.

<br>

Profiles ensure your local environment is as lightweight or as comprehensive as you need it to be at any given moment.

<br>

## 4. Persistent Data Across Container Restarts
Nothing is more frustrating than populating a local database with test data, only to lose it all when you restart your containers. 

<br>

This configuration makes extensive use of Docker Volumes (`mongodb-data`, `redis-data`, `postgresql-data`, etc.). Even if you destroy your containers (`docker compose down`), your data remains safely stored on your host machine. The next time you spin the infrastructure up, your test users, product catalogs, and AI embeddings will be right where you left them.

<br>

## 5. Consistent Development Environment Across Teams
"It works on my machine" is a phrase that should be banished from modern engineering. When teams rely on manual local installations, discrepancies in database versions, OS-level dependencies, and missing configurations inevitably lead to broken builds.

<br>

By committing this `docker-compose.yml` to your repository, you guarantee absolute consistency. Whether a developer is on a Mac, Windows, or Linux machine, running `docker compose up -d` ensures they are running:
*   The exact same version of OpenSearch (`2.19.1`)
*   The exact same configuration for MongoDB (`mongo:8`)
*   The identical pgvector extension for PostgreSQL

<br>

This dramatically reduces onboarding time for new hires. A new developer simply clones the repo, runs one command, and their local infrastructure is perfectly mirrored to the rest of the team.

<br>

**Getting Started:**

<br>

Using this setup is incredibly straightforward. First, clone the repository:

```bash
git clone https://github.com/itsujji/local-dev-infrastructure.git
cd local-dev-infrastructure
```

<br>

**Start the core infrastructure:**
```bash
docker compose up -d
```

**Start the core infrastructure AND the AI profile:**
```bash
docker compose --profile ai up -d
```

**Start EVERYTHING (Core, AI, Database, and Monitoring):**
```bash
docker compose --profile ai --profile database --profile monitoring up -d
```

**Stop everything safely (preserving data):**
```bash
docker compose --profile ai --profile database --profile monitoring down
```

**Conclusion:**
Building a reliable local infrastructure doesn't have to be a chore. By leveraging Docker Compose with advanced features like profiles, volume mapping, and comprehensive healthchecks, you can create a local environment that is powerful, modular, and flawlessly consistent. 

Adopt this setup, and watch your team spend less time fighting local environments and more time shipping great features!
