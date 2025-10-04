# CDCwithDebezium
Change data capture with Debezium and kafka connect
# Postgres CDC with Debezium (v3.1) — Complete Tutorial

This guide sets up a full CDC pipeline using **PostgreSQL → Debezium (Kafka Connect) → Kafka** with Docker Compose, then walks through producing and observing change events (insert/update/delete). An optional section shows how to add the (frozen) Debezium UI.

## What you’ll build

- PostgreSQL with **logical replication**
- Kafka + ZooKeeper (as required by the chosen Kafka image)
- Kafka Connect with **Debezium Postgres connector**
- A demo `customers` table and a topic `example.public.customers`
- (Optional) Debezium UI for form-based connector creation

---

## Prerequisites

- Docker Engine + **Docker Compose v2**
- `curl`
- Basic familiarity with Postgres & CLI

> Debezium is configured entirely via CLI / REST here (no built-in UI).

---

## Quick Start

```bash
# 0) Create a project
mkdir debezium-example && cd debezium-example

# 1) Create docker-compose.yml (see file content below)
# 2) Start the stack
docker compose up -d

# 3) Create demo table & enable full-row images
docker compose exec postgres   psql -U dbz -d example   -c "CREATE TABLE customers (id SERIAL PRIMARY KEY, name VARCHAR(255), email VARCHAR(255));"

docker compose exec postgres   psql -U dbz -d example   -c "ALTER TABLE customers REPLICA IDENTITY FULL;"

# 4) Register Debezium Postgres connector (see register-postgres.json below)
curl -X POST -H "Content-Type: application/json"      --data @register-postgres.json      http://localhost:8083/connectors

# 5) Watch CDC events
docker compose exec kafka /kafka/bin/kafka-console-consumer.sh   --bootstrap-server kafka:9092   --topic example.public.customers   --from-beginning

# 6) Produce changes in another terminal
docker compose exec postgres   psql -U dbz -d example -c "INSERT INTO customers(name,email) VALUES ('Alice','alice@example.com');"

docker compose exec postgres   psql -U dbz -d example -c "UPDATE customers SET name='Alice Updated' WHERE id=1;"

docker compose exec postgres   psql -U dbz -d example -c "DELETE FROM customers WHERE id=1;"

# 7) Clean up
docker compose down -v
```

---

## Step 1: Infrastructure (Docker Compose)

Create **`docker-compose.yml`**:

```yaml
services:
  zookeeper:
    image: quay.io/debezium/zookeeper:3.1
    ports: ["2181:2181"]

  kafka:
    image: quay.io/debezium/kafka:3.1
    depends_on: [zookeeper]
    ports: ["29092:29092"]
    environment:
      ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_LISTENERS: INTERNAL://0.0.0.0:9092,EXTERNAL://0.0.0.0:29092
      KAFKA_ADVERTISED_LISTENERS: INTERNAL://kafka:9092,EXTERNAL://localhost:29092
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: INTERNAL:PLAINTEXT,EXTERNAL:PLAINTEXT
      KAFKA_INTER_BROKER_LISTENER_NAME: INTERNAL
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1

  connect:
    image: quay.io/debezium/connect:3.1
    depends_on: [kafka]
    ports: ["8083:8083"]
    environment:
      BOOTSTRAP_SERVERS: kafka:9092
      GROUP_ID: 1
      CONFIG_STORAGE_TOPIC: connect_configs
      OFFSET_STORAGE_TOPIC: connect_offsets
      STATUS_STORAGE_TOPIC: connect_statuses
      KEY_CONVERTER_SCHEMAS_ENABLE: "false"
      VALUE_CONVERTER_SCHEMAS_ENABLE: "false"

  postgres:
    image: debezium/postgres:15
    ports: ["5432:5432"]
    command: postgres -c wal_level=logical -c max_wal_senders=10 -c max_replication_slots=10
    environment:
      POSTGRES_USER: dbz
      POSTGRES_PASSWORD: dbz
      POSTGRES_DB: example
```

Start & verify:

```bash
docker compose up -d
docker compose ps
```

You should see `zookeeper`, `kafka`, `connect`, `postgres` all **Up**.

---

## Step 2: Prepare PostgreSQL for CDC

### Create demo table

```bash
docker compose exec postgres   psql -U dbz -d example   -c "CREATE TABLE customers (id SERIAL PRIMARY KEY, name VARCHAR(255), email VARCHAR(255));"
```

### Enable full row images for updates/deletes

```bash
docker compose exec postgres   psql -U dbz -d example   -c "ALTER TABLE customers REPLICA IDENTITY FULL;"
```

### (Optional) Connect via a local SQL client

- Host: `localhost`
- Port: `5432`
- DB: `example`
- User: `dbz`
- Password: `dbz`
- Conn string: `postgresql://dbz:dbz@localhost:5432/example?sslmode=disable`

---

## Step 3: Create & Register the Debezium Connector

Create **`register-postgres.json`**:

```json
{
  "name": "example-connector",
  "config": {
    "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
    "database.hostname": "postgres",
    "database.port": "5432",
    "database.user": "dbz",
    "database.password": "dbz",
    "database.dbname": "example",
    "topic.prefix": "example",
    "slot.name": "example_slot",
    "publication.autocreate.mode": "filtered",
    "table.include.list": "public.customers"
  }
}
```

Register:

```bash
curl -X POST -H "Content-Type: application/json"      --data @register-postgres.json      http://localhost:8083/connectors
```

Check status:

```bash
curl -s http://localhost:8083/connectors/example-connector/status | jq
```

Troubleshoot logs (if needed):

```bash
docker compose logs -f connect
```

---

## Step 4: Generate & Observe CDC Events

### Start a Kafka consumer

```bash
docker compose exec kafka /kafka/bin/kafka-console-consumer.sh   --bootstrap-server kafka:9092   --topic example.public.customers   --from-beginning
```

### Produce changes in Postgres (new terminal)

**Insert**

```bash
docker compose exec postgres   psql -U dbz -d example   -c "INSERT INTO customers(name,email) VALUES ('Alice','alice@example.com');"
```

**Update**

```bash
docker compose exec postgres   psql -U dbz -d example   -c "UPDATE customers SET name='Alice Updated' WHERE id=1;"
```

**Delete**

```bash
docker compose exec postgres   psql -U dbz -d example   -c "DELETE FROM customers WHERE id=1;"
```

Expect Debezium “envelope” messages with:
- `op`: `"c"` (create), `"u"` (update), `"d"` (delete)
- `before` / `after` row states
- `source` metadata and `ts_ms` timestamp

---

## Step 5: Clean Up

```bash
docker compose down -v
```

---

## (Optional) Add Debezium UI

> **Note:** Debezium UI was **frozen** (Aug 2024) and maintained in limited capacity; a new management platform is being explored. The UI can still be used for basic form-based configs.

### Update `docker-compose.yml`

Add the UI service:

```yaml
  debezium-ui:
    image: quay.io/debezium/debezium-ui:3.1
    ports:
      - "8080:8080"
    environment:
      KAFKA_CONNECT_URIS: http://connect:8083
    depends_on:
      - connect
```

Enable the Connect REST extension for the UI (augment the `connect` service `environment`):

```yaml
    ENABLE_DEBEZIUM_KC_REST_EXTENSION: "true"
    CONNECT_REST_EXTENSION_CLASSES: io.debezium.kcrestextension.DebeziumConnectRestExtension,io.debezium.connector.postgresql.rest.DebeziumPostgresConnectRestExtension
```

Restart services:

```bash
docker compose down
docker compose up -d
```

Open the UI at **http://localhost:8080**.

### Create a connector via the UI

1. **Create a connector** → **PostgreSQL** → Next  
2. Enter connection properties for `postgres` (host `postgres`, port `5432`, db `example`, user `dbz`, pw `dbz`).  
3. Add table filter: include `public.customers`.  
4. **Finish** to create the connector.  
5. Use the Kafka consumer (as above), change data in Postgres, and observe messages.

> The UI does not provide deep observability into CDC events; use Connect logs and Kafka tools for insight.

---

## Architecture (at a glance)
<img width="1009" height="818" alt="image" src="https://github.com/user-attachments/assets/5799e729-3fbf-4a68-a317-786c0107c052" />


- **PostgreSQL** (logical replication) → WAL → **Debezium Postgres Connector**  
- **Kafka Connect** manages the connector lifecycle & offsets  
- **Kafka** receives table-scoped topics (e.g., `example.public.customers`)  
- Consumers subscribe to topics of interest

---

## Notes & Tips

- `REPLICA IDENTITY FULL` is required to emit **before** images on updates/deletes.
- Topic name is `<topic.prefix>.<schema>.<table>` → `example.public.customers`.
- For debugging connector issues, always check `connect` logs.

---

Happy streaming!
