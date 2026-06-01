# Developer Log: Resolving Container Network Mismatch and Timeout Failures on Ubuntu

**Date:** June 1, 2026  
**Author:** Abir  
**Project:** Hopper Roadside (`hopper-roadside`)  
**Status:** Solved

---

## 1. Executive Summary

During development on the `hopper-roadside` microservice architecture, a critical environmental disparity occurred when transitioning the workspace from a Windows machine to a native Ubuntu Linux environment. The application failed to initialize during `npm run dev`, throwing cascaded connection timeouts for Kafka brokers and unhandled execution rejections inside MongoDB queries.

This document traces the symptoms, diagnoses the root infrastructural and network causes, details the implemented solutions across the configuration layer, and serves as standard engineering documentation to prevent future multi-platform deployment regression.

---

## 2. Problem Statement & Symptoms

When running the application locally via `npm run dev` on Ubuntu, the server crashed during initialization. The runtime execution trace captured the following logs:

````bash
abir@betopia:~/work/hopper/hopper-roadside$ npm run dev

> server@1.0.0 dev
> ts-node-dev --respawn --transpile-only --poll ./src/server.ts

[INFO] 12:53:31 ts-node-dev ver. 2.0.0 (using ts-node ver. 10.9.2, typescript ver. 5.5.4)
...
✅ Connected to Redis
(node:847297) TimeoutNegativeWarning: -1780296812547 is a negative number.
...
{"level":"INFO","timestamp":"2026-06-01T06:53:35.578Z","logger":"kafkajs","message":"[ConsumerGroup] Consumer has joined the group"...}
{"level":"ERROR","timestamp":"2026-06-01T06:53:36.585Z","logger":"kafkajs","message":"[Connection] Connection timeout","broker":"localhost:9092","clientId":"hopper-roadside"}
{"level":"ERROR","timestamp":"2026-06-01T06:53:37.873Z","logger":"kafkajs","message":"[Connection] Connection timeout","broker":"localhost:9092","clientId":"hopper-roadside"}

Error generating custom ID: MongooseError: Operation `users.findOne()` buffering timed out after 10000ms
    at Timeout.<anonymous> (/home/abir/work/hopper/hopper-roadside/node_modules/mongoose/lib/drivers/node-mongodb-native/collection.js:185:23)

👹 unhandledRejection is detected, shuting down.... Error: Failed to generate custom ID
    at /home/abir/work/hopper/hopper-roadside/src/app/utils/generateUID.ts:29:11

Core Symptoms Broken Down:
- Kafka Broker Disconnection: KafkaJS kept printing `[Connection] Connection timeout` while attempting to handshake with `localhost:9092`.
- Database Buffering Timeout: Mongoose stalled on the `users.findOne()` query inside `generateUID.ts`, indicating that no active database connection was negotiated before the execution loop.
- Cascading Failure: The unhandled promise rejection forced an immediate application kernel shutdown.

## 3. Root Cause Analysis (RCA)

### Issue A: Linux Native Docker Network vs. Windows Layer

On Windows, Docker runs inside a virtualized compatibility layer (WSL2 or Hyper-V). Docker Desktop automatically configures internal reverse proxies and network translation loops so that mapping `localhost:9092` inside a container behaves predictably from the host terminal.

On Native Linux (Ubuntu), Docker binds directly to the host kernel namespace. The explicit setting in our old docker-compose.yml:

```yaml
KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092
````

caused Kafka to broadcast its metadata telling clients to access it via localhost. From Kafka's perspective inside its isolated Docker network bridge (`hopper-net`), `localhost` resolved to its own container. When our Node app running directly on the Ubuntu host machine (`abir@betopia`) tried to consume this metadata, the routing broke because the interface boundaries were strictly isolated.

### Issue B: MongoDB Atlas Firewalls & Mongoose Buffering Lifecycle

The application utilizes a cloud-hosted MongoDB Atlas cluster. A status check confirmed that no local `mongod` service existed (Unit `mongod.service` could not be found.). The `MongooseError` occurred because the new Ubuntu development machine's public IP address was not authorized in the MongoDB Atlas Network Whitelist, silently dropping TCP packets and inducing a 10,000ms operation timeout.

Additionally, the lifecycle order in `src/server.ts` allowed database routines to fall through asynchronously before the driver confirmed a successful connection handshake.

## 4. Resolution Strategy & Practical Fixes

To resolve these faults permanently, modifications were made across both infrastructure configurations and application orchestration files.

### Step 1: Overhauling Kafka Listeners for Multi-Environment Access

We updated the kafka container block inside `docker-compose.yml` to support a dual-listener configuration. This distinguishes between internal traffic (container-to-container) and external traffic (host-to-container).

```yaml
kafka:
  image: confluentinc/cp-kafka:7.4.0
  depends_on:
    - zookeeper
  ports:
    - "9092:9092"
  environment:
    KAFKA_BROKER_ID: 1
    KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
    # Define distinct internal and external interaction spaces
    KAFKA_LISTENERS: INTERNAL://0.0.0.0:29092,EXTERNAL://0.0.0.0:9092
    KAFKA_ADVERTISED_LISTENERS: INTERNAL://kafka:29092,EXTERNAL://localhost:9092
    KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: INTERNAL:PLAINTEXT,EXTERNAL:PLAINTEXT
    KAFKA_INTER_BROKER_LISTENER_NAME: INTERNAL
    KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
    KAFKAJS_NO_PARTITIONER_WARNING: 1
  restart: unless-stopped
  networks:
    - hopper-net
```

`INTERNAL://kafka:29092`: Directs traffic inside the `hopper-net` network using the explicit DNS name of the service.

`EXTERNAL://localhost:9092`: Routes interactions safely out of the virtual switch interface back to our native Ubuntu host development environment.

### Step 2: Whitelisting Public IP and Fixing the Initialization Pipeline

Network Access Whitelist: Logged into MongoDB Atlas and updated the Network Access rules to accept the current public IP of the Ubuntu workstation.

Code Hardening (`src/server.ts`): Refactored the entrypoint lifecycle to execute sequentially, preventing any seeding or server boots if a required service is disconnected:

```typescript
// Enforce mongoose to fail fast instead of queueing functions endlessly
mongoose.set("bufferCommands", false);

async function main() {
  try {
    console.log("Connecting to Database...");
    await mongoose.connect(config.database_url as string);
    console.log("✅ Database connected successfully");

    // Run Kafka handshakes sequentially to confirm network availability
    console.log("Connecting Kafka Services...");
    await connectKafkaAdmin();
    await connectKafkaProducer();
    await connectKafkaConsumer();

    // Start network sockets and HTTP channels only after dependencies are live
    server = app.listen(config.port, () => {
      console.log(
        colors.green(
          `Server is running successfully ${config.ip}:${config.port}`,
        ).bold,
      );
    });

    // Seed execution safely encapsulated within the initialized container context
    await seedAdmin();
    startScheduler();
  } catch (error: any) {
    console.error("❌ Server failed to start securely:", error);
    process.exit(1);
  }
}
```

## 5. Summary of Runtime Environment Context

Depending on how you run the application code, ensure your environment files conform to the following target references:

| Running Application                    | Kafka Broker URL | MongoDB String            |
| -------------------------------------- | ---------------- | ------------------------- |
| Local Terminal (`npm run dev`)         | `localhost:9092` | MongoDB Atlas Cluster URI |
| Docker Container (`docker compose up`) | `kafka:29092`    | MongoDB Atlas Cluster URI |

```

```
