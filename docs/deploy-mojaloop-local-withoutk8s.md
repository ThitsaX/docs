# Local Deployment Guide on Mojaloop Platform Components


This document reflects the local non-Kubernetes deployment maintained under Project:

👉 [Mojaloop Local Deployment Repository](https://github.com/ThitsaX/mojaloop-local-deployment)

 ⚠️ **Scope Notice**

 This deployment guide is intended for **local development, testing, and learning purposes only**.
 It is not production-hardened and does not include:

 - High availability (HA)
 - Horizontal scaling
 - TLS/mTLS enforcement
 - Network isolation
 - Observability stack (Prometheus, Grafana, etc.)

 For production deployments, refer to the Platform Architecture and Security Architecture documentation.

## Introduction

[Mojaloop](https://mojaloop.io) is an open-source platform that provides interoperable digital payment infrastructure. It was created by the Mojaloop Foundation to address the challenge of siloed payment systems that limit financial inclusion, particularly in emerging markets and underbanked regions. By offering a set of open standards and software, Mojaloop enables different financial service providers (FSPs) to connect and transact seamlessly — lowering costs and expanding access to digital financial services.

This repository, which is part of Project **MELODY**, provides a guide and configuration for deploying Mojaloop's core platform services locally, without Kubernetes or Helm. It is intended for development, testing, and learning purposes.

## Platform Components

The core Mojaloop services included in this deployment are located in the `platform/` folder. Each subfolder contains the source for a specific version/tag of the component.

| Component | Version | Description |
|---|---|---|
| [account-lookup-service](https://github.com/mojaloop/account-lookup-service) | `17.14.4` | Handles participant discovery and routing. Maintains a registry of FSPs and their endpoints so the switch can locate the correct recipient institution for a payment. |
| [quoting-service](https://github.com/mojaloop/quoting-service) | `17.13.8` | Calculates fees and FSP commissions for interoperable transactions, giving both payer and payee FSPs a complete view of all costs involved. |
| [central-ledger](https://github.com/mojaloop/central-ledger) | `19.11.3` | The core ledger that manages transfers between participating institutions. Provides real-time clearing messages, maintains account positions, and handles scheme-level fees. |
| [central-settlement](https://github.com/mojaloop/central-settlement) | `17.2.6` | Manages settlement between FSPs and the central hub. Administers settlement windows, event triggers, and FSP account details. |
| [ml-api-adapter](https://github.com/mojaloop/ml-api-adapter) | `16.6.1` | A translation layer that converts between the Mojaloop API and the internal format used by the Central Services stack. |
| [sdk-scheme-adapter](https://github.com/mojaloop/sdk-scheme-adapter) | `0.0.70` | Bridges non-Mojaloop-compliant DFSP backends with the Mojaloop switch by translating synchronous HTTP calls into native Mojaloop API interactions. |

## Infrastructure Tools

Mojaloop's platform services depend on several infrastructure tools. Configuration and scripts for each are provided in the `tools/` folder.

| Tool | Folder | Purpose |
|---|---|---|
| Apache Kafka | `tools/kafka/` | Message broker for asynchronous event streaming between services. |
| Redis | `tools/redis/` | In-memory data store used as a distributed cache (runs as a 6-node cluster). |
| MySQL | `tools/mysql/` | Relational database for central-ledger, quoting-service, and account-lookup-service. |
| Redpanda | `tools/redpanda/` | Kafka-compatible streaming platform (alternative to Apache Kafka). |

### Additional Tools

| Tool | Folder | Purpose |
|---|---|---|
| Postman | `tools/postman/` | API collection (`Melody.postman_collection.json`) for testing Mojaloop endpoints. |
| JMeter | `tools/jmeter/` | Performance/load testing scripts for wallet-to-wallet transfers. |

## Setup Guide

### 1. Kafka

Mojaloop uses Apache Kafka (in KRaft mode — no ZooKeeper) for event streaming between services. Download Kafka from https://dlcdn.apache.org/kafka/3.9.1/kafka_2.12-3.9.1.tgz.

#### 1.1 Configure

Copy the provided `server.properties` into your Kafka installation:

```bash
cp tools/kafka/server.properties <KAFKA_HOME>/config/server.properties
```

Key settings in the provided configuration:

| Setting | Value | Notes |
|---|---|---|
| `process.roles` | `broker,controller` | Runs in KRaft mode (combined broker + controller) |
| `listeners` | `INTERNAL://0.0.0.0:9092, EXTERNAL://0.0.0.0:9094, CONTROLLER://localhost:9093` | Internal for local services, External for Docker containers |
| `advertised.listeners` | `INTERNAL://localhost:9092, EXTERNAL://host.docker.internal:9094` | Docker services connect via `host.docker.internal:9094` |
| `log.dirs` | `/home/mojaloop/storage` | Update this to your preferred data directory |

#### 1.2 Initialize and Start

```bash
# From your Kafka installation directory:

# Initialize the storage (first time only)
./tools/kafka/scripts/init-kafka.sh

# Start the broker
./tools/kafka/scripts/start-kafka.sh
```

#### 1.3 Create Topics

Mojaloop requires a set of predefined Kafka topics. Create them with:

```bash
./tools/kafka/scripts/create-topics.sh [KAFKA_HOST] [KAFKA_PORT]
# Defaults to localhost:9092 if no arguments given
```

This creates the following topics (replication-factor=1, partitions=1):

| Category | Topics |
|---|---|
| Transfers | `topic-transfer-prepare`, `topic-transfer-position`, `topic-transfer-fulfil`, `topic-transfer-get`, `topic-transfer-position-batch` |
| Admin | `topic-admin-transfer` |
| Notifications | `topic-notification-event` |
| Bulk | `topic-bulk-prepare`, `topic-bulk-fulfil`, `topic-bulk-processing`, `topic-bulk-get` |
| Quotes | `topic-quotes-post`, `topic-quotes-put`, `topic-quotes-get` |
| FX Quotes | `topic-fx-quotes-post`, `topic-fx-quotes-put`, `topic-fx-quotes-get` |
| Bulk Quotes | `topic-bulkquotes-post`, `topic-bulkquotes-put`, `topic-bulkquotes-get` |
| Settlement | `topic-deferredsettlement-close` |

#### 1.4 Utility Scripts

| Script | Purpose |
|---|---|
| `tools/kafka/scripts/list-group-details.sh` | Lists consumer group lag and offsets for all Mojaloop consumer groups |
| `tools/kafka/scripts/remove-all-topics.sh` | Deletes all non-internal topics (useful for a clean reset) |

#### 1.5 Redpanda Console (Optional)

For a web UI to inspect Kafka topics, consumer groups, and messages, you can run the Redpanda Console:

```bash
docker compose -f tools/redpanda/docker-compose.yml up -d
```

The console will be available at http://localhost:18080. It connects to Kafka via `host.docker.internal:9094` (the EXTERNAL listener).

---

### 2. Redis Cluster

Mojaloop uses a Redis cluster as a distributed cache. The provided script sets up a 3-node cluster locally.

**Prerequisites:** Redis **7** must be installed. Ensure `redis-server` and `redis-cli` are available on your `PATH`.

#### 2.1 Start the Cluster

```bash
cd tools/redis
./start-redis-cluster.sh
```

This will:
1. Create a `redis-cluster/` directory with per-node configuration and data
2. Start 3 Redis instances on ports **6379**, **6380**, and **6381**
3. Form them into a cluster with `--cluster-replicas 0` (no replicas, masters only)

Each node runs with `cluster-enabled yes`, `appendonly yes`, and `protected-mode no`.

#### 2.2 Stop the Cluster

```bash
cd tools/redis
./stop-redis-cluster.sh
```

This shuts down all 3 nodes (ports 6379–6381) gracefully.

---

### 3. MySQL

Mojaloop's central-ledger, quoting-service, and account-lookup-service store data in MySQL. Version **8.4** or lower is supported (Mojaloop recommends **5.7**).

#### 3.1 Configure

Use the provided `my.cnf` as a template:

```ini
[mysqld]
basedir=/<your_mysql_location>
datadir=/<your_mysql_location>/data
socket=/tmp/mysql.sock
port=3306
mysql_native_password=ON
```

> **Important:** If using MySQL 8.4+, the `mysql_native_password=ON` setting is required because Mojaloop's database libraries rely on the `mysql_native_password` authentication plugin.

#### 3.2 Create the Database User

```sql
-- Create the user (or alter if it already exists)
ALTER USER 'central_ledger'@'%' IDENTIFIED WITH mysql_native_password BY 'password';

-- Grant full privileges (required for Knex DB migrations)
GRANT ALL PRIVILEGES ON *.* TO 'central_ledger'@'%';
FLUSH PRIVILEGES;
```

`central_ledger` is the default username used by Mojaloop services. You can change it, but you will need to update the service configurations accordingly.

---

### 4. Running Platform Services

> **Prerequisites:** Ensure Kafka, Redis, and MySQL are running before starting any platform service.

Each service is pre-configured with a `default.json` that points to the local infrastructure tools (Kafka at `localhost:9092`, Redis at `localhost:6379`, MySQL at `localhost:3306`). If you are using the same settings from the setup steps above, no configuration changes are needed.

Services should be started in the following order:

#### 4.1 Central Ledger

The central-ledger is the core service and must be started first, as other services depend on it.

```bash
cd platform/central-ledger-19.11.3
npm install
npm run start:api
```

| Setting | Value |
|---|---|
| API Port | `3001` |
| MySQL User / Schema | `central_ledger` / `central_ledger` |
| Config | `platform/central-ledger-19.11.3/config/default.json` |

> On first run, Knex will automatically run database migrations (`MIGRATIONS.DISABLED: false`).

#### 4.2 Account Lookup Service

```bash
cd platform/account-lookup-service-17.14.4
npm install
npm run start:all
```

| Setting | Value |
|---|---|
| Admin Port | `4001` |
| API Port | `4002` |
| Monitoring Port | `4003` |
| MySQL User / Schema | `account_lookup` / `account_lookup` |
| Config | `platform/account-lookup-service-17.14.4/config/default.json` |

> This service uses a separate MySQL user (`account_lookup`). Make sure to create it the same way as `central_ledger` (see [Section 3.2](#32-create-the-database-user)).

#### 4.3 Quoting Service (API)

```bash
cd platform/quoting-service-17.13.8
npm install
npm run start:api
```

| Setting | Value |
|---|---|
| API Port | `3002` |
| Monitoring Port | `3003` |
| MySQL User / Schema | `central_ledger` / `central_ledger` |
| Config | `platform/quoting-service-17.13.8/config/default.json` |

#### 4.4 Quoting Service (Handlers)

In a **separate terminal**, start the Kafka consumer handlers for the quoting service:

```bash
cd platform/quoting-service-17.13.8
npm run start:handlers
```

> The handlers consume from Kafka topics (`topic-quotes-*`, `topic-bulkquotes-*`, `topic-fx-quotes-*`) and process quote requests asynchronously. `npm install` is not needed again if already done in step 4.3.

#### 4.5 ML API Adapter

```bash
cd platform/ml-api-adapter-16.6.1
npm install
npm run start:api
```

| Setting | Value |
|---|---|
| API Port | `3000` |
| Central Ledger Endpoint | `http://localhost:3001` |
| Config | `platform/ml-api-adapter-16.6.1/config/default.json` |

> This is the main entry point for FSPIOP API calls. It forwards transfer requests to the central-ledger via Kafka.

#### 4.6 Central Settlement

```bash
cd platform/central-settlement-17.2.6
npm install
npm run start:api
```

| Setting | Value |
|---|---|
| API Port | `3007` |
| MySQL User / Schema | `central_ledger` / `central_ledger` |
| Central Ledger Endpoint | `http://localhost:3001` |
| Config | `platform/central-settlement-17.2.6/config/default.json` |

---

### 5. Building and Running the Simulator

The simulator provides two wallet instances (wallet1 and wallet2) that act as DFSPs, allowing you to test end-to-end transfers through the Mojaloop platform.

The simulator's **Connector** component is production-grade and can be used as the foundation for connecting your FSP to Mojaloop. It reduces one network hop compared to the standard approach:

| Approach | Flow | Hops to Hub |
|---|---|---|
| **Connector** (recommended) | FSP → Connector → Hub | 2 |
| sdk-scheme-adapter | FSP → Core Connector → sdk-scheme-adapter → Hub | 3 |

Both approaches work, but if you want a leaner integration, the Connector provides a more direct path. Alternatively, you can use the [sdk-scheme-adapter](https://github.com/mojaloop/sdk-scheme-adapter) with a [Core Connector](https://docs.mojaloop.io/mojaloop-core-connector-specification/) if that better suits your architecture.

> **Note:** If you need ready-made features such as mTLS, transaction logging, and a management portal, consider using [PM4ML (Payment Manager for Mojaloop)](https://github.com/pm4ml), which already bundles a Core Connector and sdk-scheme-adapter out of the box.

**Prerequisites:** Apache Maven and **JDK 21** (or 25) must be installed.

#### 5.1 Build the Simulator

```bash
cd simulator
mvn install -DskipTests=true
```

#### 5.2 Prepare the Wallets

After the build completes, copy the connector JAR and its dependencies into the `wallets/` folder:

```bash
mkdir -p wallets/lib
cp simulator/connector/target/mod_connector-1.0.0.jar wallets/
cp simulator/connector/target/lib/* wallets/lib/
```

Your `wallets/` folder should look like this:

```
wallets/
├── mod_connector-1.0.0.jar
├── lib/                          # dependency JARs
├── wallet1.sh
├── wallet1-log4j2.xml
├── wallet2.sh
└── wallet2-log4j2.xml
```

#### 5.3 Run the Wallets

Start each wallet in a **separate terminal**:

```bash
# Terminal 1 — Wallet 1
cd wallets
./wallet1.sh
```

```bash
# Terminal 2 — Wallet 2
cd wallets
./wallet2.sh
```

| Wallet | FSP ID | Inbound Port | Outbound Port |
|---|---|---|---|
| Wallet 1 | `wallet1` | `8081` | `8080` |
| Wallet 2 | `wallet2` | `9091` | `9090` |

Both wallets connect to the platform services at their default local ports (account-lookup on `4002`, quoting-service on `3002`, ml-api-adapter on `3000`).

---

### 6. Onboarding Wallets

Once all platform services and wallets are running, you need to onboard the wallets (DFSPs) onto the Mojaloop hub. This is done using the Postman collection at `tools/postman/Melody.postman_collection.json`.

**Prerequisites:** Import the collection into [Postman](https://www.postman.com/).

#### 6.1 Setup the Hub

Open the **onboarding > create-hub** folder in the Postman collection and run each request in order:

1. `list enums` — verify the hub is reachable
2. `setup-HUB_MULTILATERAL_SETTLEMENT` — create the multilateral settlement account
3. `setup-HUB_RECONCILIATION` — create the reconciliation account
4. `create settlement model` — configure the settlement model
5. `add Oracle` *(optional)* — register the an oracle for party lookups. Skip this if you do not have Oracle.
6. `get Oracles` *(optional)* — verify the oracle was registered

#### 6.2 Create Participants

Open the **onboarding > create-participants** folder. Run the requests for **wallet1** and **wallet2** (steps 1–4 in order):

For each wallet:

| Step | Request | Purpose |
|---|---|---|
| 1 | `create new participant` | Register the DFSP with the hub |
| 2 | `activate POSITION account` | Activate the position account |
| 3 | `create limit and position` | Set the Net Debit Cap (NDC) limit |
| 4 | `fund In to SETTLEMENT account` | Fund the settlement account |

> After creating participants, proceed to [Section 6.3](#63-register-wallet-endpoints) to register the callback endpoints for wallet1 and wallet2.

#### 6.3 Register Wallet Endpoints

After completing the Postman steps above, run the endpoint registration scripts to register all FSPIOP callback URLs for each wallet:

```bash
./onboarding/register-wallet1-endpoints.sh
./onboarding/register-wallet2-endpoints.sh
```

These scripts register callback endpoints (parties, quotes, transfers, bulk operations, etc.) against the central-ledger, pointing to each wallet's outbound port:

| Wallet | Callback Base URL |
|---|---|
| Wallet 1 | `http://localhost:8080` |
| Wallet 2 | `http://localhost:9090` |

After this step, both wallets are fully onboarded and ready to send/receive transfers through the Mojaloop platform.

---

### 7. Performing Transfers

With all services running and wallets onboarded, you can perform end-to-end transfers using Apache JMeter.

**Prerequisites:** **JDK 21** (or 25) and [Apache JMeter **5.6.3**](https://jmeter.apache.org/download_jmeter.cgi) must be installed.

#### 7.1 Run a Transfer Test

1. Open Apache JMeter
2. Load the test plan: **File > Open** → `tools/jmeter/wallet1-to-wallet2.jmx`
3. Click the **Start** button (▶) to run the test

The test plan sends transfers from **wallet1** to **wallet2** through the full Mojaloop flow (party lookup → quote → transfer). By default it runs with 1 thread and 100 iterations.

| Setting | Value |
|---|---|
| Payer FSP | `wallet1` |
| Payee FSP | `wallet2` |
| Target | `http://localhost:8081` (wallet1 inbound) |
| Threads | `1` |
| Loop Count | `100` |

> A reverse test plan (`wallet2-to-wallet1.jmx`) is also available in the same folder for testing transfers in the opposite direction.

---

### 8. Settlement

After performing transfers, you can settle the net positions between participants. The settlement process is available in the Postman collection under the **settlement** folder.

Follow these steps in order:

#### 8.1 Close the Current Settlement Window

Close the currently open settlement window so that all completed transfers are captured for settlement.

#### 8.2 Create the Settlement

Create a new settlement to calculate the net amounts to be settled between participants. This will show how much each FSP owes or is owed based on the transfers that occurred within the closed window.

#### 8.3 Update the Settlement Status

Update the settlement status to progress it through the settlement lifecycle (e.g., from `PENDING_SETTLEMENT` to `PS_TRANSFERS_RECORDED` to `PS_TRANSFERS_COMMITTED`).

#### 8.4 Deposit or Withdraw Funds

Based on the net settlement amounts from step 8.2:
- **Deposit** funds into the settlement account for FSPs that owe money (net debtors)
- **Withdraw** funds from the settlement account for FSPs that are owed money (net creditors)

This final step reconciles the actual fund movements and completes the settlement cycle.

---

### Port Summary

| Service | Port(s) |
|---|---|
| ml-api-adapter | `3000` |
| central-ledger | `3001` |
| quoting-service (API) | `3002` |
| quoting-service (Monitoring) | `3003` |
| central-settlement | `3007` |
| account-lookup-service (Admin) | `4001` |
| account-lookup-service (API) | `4002` |
| account-lookup-service (Monitoring) | `4003` |
| Wallet 1 (Inbound / Outbound) | `8081` / `8080` |
| Wallet 2 (Inbound / Outbound) | `9091` / `9090` |
