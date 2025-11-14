# Kubernetes-voting-app
Kubernetes voting app. This file is nothing more than a hello world project for me to learn some basic kubernetes. 

This project is a hello-world Kubernetes exercise used to learn the fundamentals of Pods, Services, and Deployments.
It is structured in two parts:

``Pods`` version – each component is created manually as an individual Pod, purely for learning.

``Deployments`` version – a more realistic setup using Deployments to manage Pods automatically.

This README covers concepts common to both versions: architecture, communication, and how Services connect the application together. 

redis service metadata name is important as it allows you to get by name not IP
redis service taregt port relates to the redis pod container port, which for redis is typically 6379. The service port can be anything but has been chosen same as container port for simplicity. 
Service type for db-service and redis-service is set as ClusterIP. However, this is the default type and can be ommitted. 
the selector in the redis service is based on the label in the redis pod, thats how the service knows which pod to act upon. 

## Key Concepts
### Service discovery via names
Kubernetes Services can be reached by DNS name, not IP address.
For example, the Redis Service is reachable at:
```
redis-service
```
This works because:
The Service metadata name defines the DNS name. Kubernetes automatically keeps this stable, even when Pod IPs change.


### Ports & targeting
For Redis:
- ``Container port``: 6379 (standard Redis port)
- ``Service targetPort``: must match the container port
- ``Service port``: can be anything, usually kept the same for simplicity


### ClusterIP is the default

``redis-service`` and ``db-service`` both use ``ClusterIP``, which:
- is the default Service type
- exposes the Service inside the cluster only
- does not allow external traffic


No need to specify it unless you want clarity.


### Labels and selectors

Services route traffic using labels.
Example:
- Redis Pod has a label: ``app: redis``
- Redis Service has a selector: ``app: redis``

The selector is how Kubernetes knows which Pod(s) the Service should send traffic to.

## Architecture
```
                          ┌────────────────────────┐
                          │        USERS           │
                          └────────────┬───────────┘
                                       │
                                       ▼
                             ┌──────────────────┐
                             │  vote Service    │
                             │ (NodePort)       │
                             └────────┬─────────┘
                                      │
                                      ▼
                              ┌───────────────┐
                              │   vote Pod    │
                              └───────────────┘
                                      │
                         pushes votes to Redis queue
                                      │
                                      ▼
                             ┌──────────────────┐
                             │  redis Service   │
                             │  (ClusterIP)     │
                             └────────┬─────────┘
                                      │
                                      ▼
                              ┌───────────────┐
                              │   redis Pod   │
                              └───────────────┘
                                      │
                    worker reads from Redis, writes to DB
                                      │
                                      ▼
                              ┌───────────────┐
                              │  worker Pod   │
                              └───────────────┘
                                      │
                                      ▼
                             ┌──────────────────┐
                             │   db Service     │
                             │  (ClusterIP)     │
                             └────────┬─────────┘
                                      │
                                      ▼
                              ┌───────────────┐
                              │    db Pod     │
                              └───────────────┘
                                      │
                         result service reads from DB
                                      │
                                      ▼
                             ┌──────────────────┐
                             │ result Service   │
                             │   (NodePort)     │
                             └────────┬─────────┘
                                      │
                                      ▼
                              ┌───────────────┐
                              │  result Pod   │
                              └───────────────┘
                                      │
                                      ▼
                          ┌────────────────────────┐
                          │        USERS           │
                          └────────────────────────┘

```

| Component  | What it does                                                    | Exposed?   | Why it needs a Service                                     |
| ---------- | --------------------------------------------------------------- | ---------- | ---------------------------------------------------------- |
| **vote**   | The web frontend for users to submit votes                      | ✅ External | So users (outside cluster) can access it                   |
| **result** | Web frontend to display vote results                            | ✅ External | So users can view it                                       |
| **redis**  | Temporary queue for storing unprocessed votes                   | ❌ Internal | Only `vote` and `worker` pods need to reach it             |
| **worker** | Background job processor — takes votes from Redis, writes to DB | ❌ No       | It connects *out* to Redis and DB; no one connects *to* it |
| **db**     | PostgreSQL database storing final vote data                     | ❌ Internal | Only `worker` and `result` need to reach it                |




## Services and Workers
`services` connect two components of the applicatiom:
- The vote frontend needs to talk to redis. Instead of hardcoding a pod IP (which changes constantly), it just sends requests to redis-service.
- Kubernetes then automatically routes that request to one of the running redis pods.

`worker` isn’t a Service because it doesn’t need incoming traffic. It’s a “background engine” that just processes jobs in the queue:
- The vote frontend takes user votes and pushes them into Redis (a fast in-memory queue).
- The worker constantly watches that Redis queue, picks up votes, and writes them into the Postgres DB.
- It’s essentially the bridge between Redis and Postgres.

### Why Services are needed
Services provide stable internal networking so components can talk to each other without relying on Pod IPs (which constantly change).

Examples:
- The vote frontend sends votes to redis-service.
- The worker reads from redis-service and writes to db-service.
- The result frontend reads from db-service.

### Why the worker does NOT get a Service
The worker is a background processor:
- It does not receive traffic.
- It only connects out to Redis and the database.

So a Service would be pointless.

### Flow summary:
- Frontend (vote) → pushes vote into Redis
- Worker → pulls vote from Redis → saves to DB

```
vote Pod → redis-service → worker Pod → db-service → result Pod
```