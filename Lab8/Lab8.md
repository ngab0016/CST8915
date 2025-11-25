# Lab8 - ngab0016 - 041196196

## Demo video
- Youtube video link: https://youtu.be/7i70WhLDcnI

## Task 2 - Solution

# Task 2: Production-Ready Improvements - Solution Summary

## Problem Statement

The initial deployment had two critical issues that compromised data integrity and service reliability:

1.  **Data Loss:** MongoDB and RabbitMQ stored data inside containers, meaning all application state and queued messages were permanently lost upon pod restarts or failure.
2.  **No High Availability (HA):** A single MongoDB instance created a single point of failure and if that pod failed, the entire ordering system would be disabled.

---

## Solution 1: MongoDB Replica Set with Persistent Storage

To solve the high availability and data persistence issues for the database, MongoDB was reconfigured as a 3-node replica set managed by a **StatefulSet**.

### High Availability and Architecture

| Feature | Implementation Details | Benefit |
| :--- | :--- | :--- |
| **Replica Set (HA)** | Deployed as a **3-node replica set** (`rs0`). Each pod runs with the `--replSet rs0` flag. | If the **PRIMARY** node fails, a new PRIMARY is automatically elected from the two **SECONDARY** nodes, ensuring continuous database writes and **zero downtime**.  |
| **Service Discovery** | Configured a **Headless Service** (`clusterIP: None`). | Provides stable, predictable DNS names (`mongodb-0.mongodb`, etc.), which is required for replica set members to discover and communicate with each other. |
| **Initialization** | The replica set must be **manually initialized** after deployment using `mongors.initiate()` on the first node (`mongodb-0`). | Establishes the set membership and begins replication. |

### Data Persistence

| Component | Implementation Details | Benefit |
| :--- | :--- | :--- |
| **Persistent Storage** | Each MongoDB pod utilizes a dedicated **10Gi PersistentVolumeClaim (PVC)**. | Data, mounted to `/data/db`, **persists** across pod restarts, pod deletions, and node failures, eliminating the risk of database data loss. |
| **Connection String** | The application connection string (`ORDER_DB_URI`) was updated to include all three hostnames and the replica set parameter: `mongodb://admin:password@mongodb-0.mongodb:27017,mongodb-1.mongodb:27017,mongodb-2.mongodb:27017/?replicaSet=rs0`. | Ensures the application can connect and handle failovers transparently. |

---

## Solution 2: RabbitMQ Persistent Storage

To ensure the durability of messages and queue definitions, persistent storage was added to the RabbitMQ deployment.

| Feature | Implementation Details | Benefit |
| :--- | :--- | :--- |
| **Persistence** | Added a dedicated **5Gi PersistentVolumeClaim (PVC)** to the RabbitMQ StatefulSet. | **Messages and queue definitions** (the core data) survives when pod restarts or crashes, eliminating message **data loss** and preserving the state of the message queue as seen in the demo video. |
| **Volume Mounting** | The PVC is mounted at the official RabbitMQ data directory: `/var/lib/rabbitmq`. | Ensures user accounts, virtual hosts and queue state are preserved. |
| **ConfigMap** | The existing ConfigMap was maintained and mounted at `/etc/rabbitmq`. | Correctly separates dynamic persistent data from static configuration files. |