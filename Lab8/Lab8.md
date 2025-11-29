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

## Updated yaml file for Lab8:
`
# ------------------------------------------------------
# Headless Service for MongoDB StatefulSet (Replica Set)
# ------------------------------------------------------
apiVersion: v1
kind: Service
metadata:
  name: mongodb
spec:
  clusterIP: None              # Headless service for stable DNS
  selector:
    app: mongodb
  ports:
    - name: mongodb
      port: 27017
      targetPort: 27017
---
# ------------------------------------------------------
# StatefulSet for MongoDB Replica Set (3 nodes)
# ------------------------------------------------------
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mongodb
spec:
  serviceName: mongodb         
  replicas: 3                  # High availability replica set
  selector:
    matchLabels:
      app: mongodb
  template:
    metadata:
      labels:
        app: mongodb
    spec:
      nodeSelector:
        "kubernetes.io/os": linux
      containers:
      - name: mongodb
        image: mongo:4.2
        ports:
          - containerPort: 27017
        env:
        - name: MONGO_INITDB_ROOT_USERNAME
          value: "admin"
        - name: MONGO_INITDB_ROOT_PASSWORD
          value: "password"
        command:
        - mongod
        - "--replSet"
        - "rs0"
        - "--bind_ip_all"
        volumeMounts:
        - name: mongodb-data
          mountPath: /data/db
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 512Mi
  volumeClaimTemplates:
  - metadata:
      name: mongodb-data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 10Gi
---
# ------------------------------------------------------
# RabbitMQ StatefulSet (with persistent storage + config)
# ------------------------------------------------------
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: rabbitmq
spec:
  serviceName: rabbitmq
  replicas: 1
  selector:
    matchLabels:
      app: rabbitmq
  template:
    metadata:
      labels:
        app: rabbitmq
    spec:
      nodeSelector:
        "kubernetes.io/os": linux
      containers:
      - name: rabbitmq
        image: rabbitmq:3-management
        ports:
          - containerPort: 5672
            name: amqp
          - containerPort: 15672
            name: management
        env:
        - name: RABBITMQ_DEFAULT_USER
          value: "username"
        - name: RABBITMQ_DEFAULT_PASS
          value: "password"
        volumeMounts:
          - name: rabbitmq-data
            mountPath: /var/lib/rabbitmq
          - name: rabbitmq-config
            mountPath: /etc/rabbitmq
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 512Mi
      volumes:
      - name: rabbitmq-config
        configMap:
          name: rabbitmq-config
  volumeClaimTemplates:
  - metadata:
      name: rabbitmq-data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 5Gi
---
# Service for RabbitMQ
apiVersion: v1
kind: Service
metadata:
  name: rabbitmq
spec:
  selector:
    app: rabbitmq
  ports:
    - name: rabbitmq-amqp
      port: 5672
      targetPort: 5672
    - name: rabbitmq-http
      port: 15672
      targetPort: 15672
  type: ClusterIP
---
# Deployment for Order Service
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
spec:
  replicas: 1
  selector:
    matchLabels:
      app: order-service
  template:
    metadata:
      labels:
        app: order-service
    spec:
      nodeSelector:
        "kubernetes.io/os": linux
      containers:
        - name: order-service
          image: darkxnight/order-service:v1
          ports:
            - containerPort: 3000
          env:
            - name: ORDER_QUEUE_HOSTNAME
              value: "rabbitmq"
            - name: ORDER_QUEUE_PORT
              value: "5672"
            - name: ORDER_QUEUE_USERNAME
              value: "username"
            - name: ORDER_QUEUE_PASSWORD
              value: "password"
            - name: ORDER_QUEUE_NAME
              value: "orders"
            - name: FASTIFY_ADDRESS
              value: "0.0.0.0"
          resources:
            requests:
              cpu: 1m
              memory: 50Mi
            limits:
              cpu: 100m
              memory: 256Mi
          startupProbe:
            httpGet:
              path: /health
              port: 3000
            failureThreshold: 5
            initialDelaySeconds: 20
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /health
              port: 3000
            failureThreshold: 3
            initialDelaySeconds: 3
            periodSeconds: 5
          livenessProbe:
            httpGet:
              path: /health
              port: 3000
            failureThreshold: 5
            initialDelaySeconds: 3
            periodSeconds: 3
      initContainers:
        - name: wait-for-rabbitmq
          image: busybox
          command:
            [
              "sh",
              "-c",
              "until nc -zv rabbitmq 5672; do echo waiting for rabbitmq; sleep 2; done;",
            ]
          resources:
            requests:
              cpu: 1m
              memory: 50Mi
            limits:
              cpu: 100m
              memory: 256Mi
---
# Service for Order Service
apiVersion: v1
kind: Service
metadata:
  name: order-service
spec:
  type: ClusterIP
  ports:
    - name: http
      port: 3000
      targetPort: 3000
  selector:
    app: order-service
---
# Deployment for Makeline Service
apiVersion: apps/v1
kind: Deployment
metadata:
  name: makeline-service
spec:
  replicas: 1
  selector:
    matchLabels:
      app: makeline-service
  template:
    metadata:
      labels:
        app: makeline-service
    spec:
      nodeSelector:
        "kubernetes.io/os": linux
      containers:
        - name: makeline-service
          image: darkxnight/makeline-service:v1
          ports:
            - containerPort: 3001
          env:
            - name: ORDER_QUEUE_URI
              value: "amqp://username:password@rabbitmq:5672"
            - name: ORDER_QUEUE_USERNAME
              value: "username"
            - name: ORDER_QUEUE_PASSWORD
              value: "password"
            - name: ORDER_QUEUE_NAME
              value: "orders"
            - name: ORDER_DB_URI
              value: "mongodb://admin:password@mongodb-0.mongodb:27017,mongodb-1.mongodb:27017,mongodb-2.mongodb:27017/?replicaSet=rs0"
            - name: ORDER_DB_NAME
              value: "orderdb"
            - name: ORDER_DB_COLLECTION_NAME
              value: "orders"
          resources:
            requests:
              cpu: 1m
              memory: 6Mi
            limits:
              cpu: 5m
              memory: 20Mi
          startupProbe:
            httpGet:
              path: /health
              port: 3001
            failureThreshold: 10
            periodSeconds: 5
          readinessProbe:
            httpGet:
              path: /health
              port: 3001
            failureThreshold: 3
            initialDelaySeconds: 3
            periodSeconds: 5
          livenessProbe:
            httpGet:
              path: /health
              port: 3001
            failureThreshold: 5
            initialDelaySeconds: 3
            periodSeconds: 3
---
# Service for Makeline Service
apiVersion: v1
kind: Service
metadata:
  name: makeline-service
spec:
  type: ClusterIP
  ports:
    - name: http
      port: 3001
      targetPort: 3001
  selector:
    app: makeline-service
---
# Deployment for Product Service
apiVersion: apps/v1
kind: Deployment
metadata:
  name: product-service
spec:
  replicas: 1
  selector:
    matchLabels:
      app: product-service
  template:
    metadata:
      labels:
        app: product-service
    spec:
      nodeSelector:
        "kubernetes.io/os": linux
      containers:
        - name: product-service
          image: darkxnight/product-service:v1
          ports:
            - containerPort: 3002
          env:
            - name: AI_SERVICE_URL
              value: "http://ai-service:5001/"
          resources:
            requests:
              cpu: 1m
              memory: 1Mi
            limits:
              cpu: 2m
              memory: 20Mi
          readinessProbe:
            httpGet:
              path: /health
              port: 3002
            failureThreshold: 3
            initialDelaySeconds: 3
            periodSeconds: 5
          livenessProbe:
            httpGet:
              path: /health
              port: 3002
            failureThreshold: 5
            initialDelaySeconds: 3
            periodSeconds: 3
---
# Service for Product Service
apiVersion: v1
kind: Service
metadata:
  name: product-service
spec:
  type: ClusterIP
  ports:
    - name: http
      port: 3002
      targetPort: 3002
  selector:
    app: product-service
---
# Deployment for Store Front
apiVersion: apps/v1
kind: Deployment
metadata:
  name: store-front
spec:
  replicas: 1
  selector:
    matchLabels:
      app: store-front
  template:
    metadata:
      labels:
        app: store-front
    spec:
      nodeSelector:
        "kubernetes.io/os": linux
      containers:
        - name: store-front
          image: darkxnight/store-front:v1
          ports:
            - containerPort: 8080
              name: store-front
          env:
            - name: VUE_APP_ORDER_SERVICE_URL
              value: "http://order-service:3000/"
            - name: VUE_APP_PRODUCT_SERVICE_URL
              value: "http://product-service:3002/"
          resources:
            requests:
              cpu: 1m
              memory: 200Mi
            limits:
              cpu: 1000m
              memory: 512Mi
          startupProbe:
            httpGet:
              path: /health
              port: 8080
            failureThreshold: 3
            initialDelaySeconds: 5
            periodSeconds: 5
          readinessProbe:
            httpGet:
              path: /health
              port: 8080
            failureThreshold: 3
            initialDelaySeconds: 3
            periodSeconds: 3
          livenessProbe:
            httpGet:
              path: /health
              port: 8080
            failureThreshold: 5
            initialDelaySeconds: 3
            periodSeconds: 3
---
# Service for Store Front
apiVersion: v1
kind: Service
metadata:
  name: store-front
spec:
  ports:
    - port: 80
      targetPort: 8080
  selector:
    app: store-front
  type: LoadBalancer
---
# Deployment for Store Admin
apiVersion: apps/v1
kind: Deployment
metadata:
  name: store-admin
spec:
  replicas: 1
  selector:
    matchLabels:
      app: store-admin
  template:
    metadata:
      labels:
        app: store-admin
    spec:
      nodeSelector:
        "kubernetes.io/os": linux
      containers:
        - name: store-admin
          image: darkxnight/store-admin:v1
          ports:
            - containerPort: 8081
              name: store-admin
          env:
            - name: VUE_APP_PRODUCT_SERVICE_URL
              value: "http://product-service:3002/"
            - name: VUE_APP_MAKELINE_SERVICE_URL
              value: "http://makeline-service:3001/"
          resources:
            requests:
              cpu: 1m
              memory: 200Mi
            limits:
              cpu: 1000m
              memory: 512Mi
          startupProbe:
            httpGet:
              path: /health
              port: 8081
            failureThreshold: 3
            initialDelaySeconds: 5
            periodSeconds: 5
          readinessProbe:
            httpGet:
              path: /health
              port: 8081
            failureThreshold: 3
            initialDelaySeconds: 3
            periodSeconds: 5
          livenessProbe:
            httpGet:
              path: /health
              port: 8081
            failureThreshold: 5
            initialDelaySeconds: 3
            periodSeconds: 3
---
# Service for Store Admin
apiVersion: v1
kind: Service
metadata:
  name: store-admin
spec:
  ports:
    - port: 80
      targetPort: 8081
  selector:
    app: store-admin
  type: LoadBalancer
---
# Deployment for AI Service
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ai-service
spec:
  replicas: 1
  selector:
    matchLabels:
      app: ai-service
  template:
    metadata:
      labels:
        app: ai-service
    spec:
      nodeSelector:
        "kubernetes.io/os": linux
      containers:
        - name: ai-service
          image: darkxnight/ai-service:v1
          ports:
            - containerPort: 5001
          env:
            - name: USE_AZURE_OPENAI
              value: "False"
            - name: AZURE_OPENAI_API_VERSION
              value: "2024-07-01-preview"
            - name: AZURE_OPENAI_GPT_DEPLOYMENT_NAME
              value: ""
            - name: AZURE_OPENAI_GPT_ENDPOINT
              value: ""
            - name: AZURE_OPENAI_DALLE_ENDPOINT
              value: ""
            - name: AZURE_OPENAI_DALLE_DEPLOYMENT_NAME
              value: ""
            - name: OPENAI_GPT_API_KEY
              valueFrom:
                secretKeyRef:
                  name: openai-gpt-api-secret
                  key: OPENAI_GPT_API_KEY
            - name: OPENAI_DALLE_API_KEY
              valueFrom:
                secretKeyRef:
                  name: openai-dalle-api-secret
                  key: OPENAI_DALLE_API_KEY
          resources:
            requests:
              cpu: 20m
              memory: 50Mi
            limits:
              cpu: 50m
              memory: 128Mi
          startupProbe:
            httpGet:
              path: /health
              port: 5001
            initialDelaySeconds: 60
            failureThreshold: 3
            periodSeconds: 5
          readinessProbe:
            httpGet:
              path: /health
              port: 5001
            initialDelaySeconds: 3
            failureThreshold: 10
            periodSeconds: 10
          livenessProbe:
            httpGet:
              path: /health
              port: 5001
            initialDelaySeconds: 3
            failureThreshold: 10
            periodSeconds: 10
---
# Service for AI Service
apiVersion: v1
kind: Service
metadata:
  name: ai-service
spec:
  type: ClusterIP
  ports:
    - name: http
      port: 5001
      targetPort: 5001
  selector:
    app: ai-service
    `
