## Product Choice

- **Product name:** Telegram
- **Website:** [telegram.org](https://telegram.org)
- **Description:** Telegram is a cloud-based instant messaging platform that focuses on speed, security, and privacy, offering features like encrypted messaging, large group chats, channels, and bots.

## Main components

![Telegram Component Diagram](./out/telegram/component-diagram/Component%20Diagram.svg)

[Telegram Component Diagram code](./src/telegram/component-diagram.puml)

### Selected Components:

1. **MTProto Gateway (DC Entry)** — The primary connection point for all Telegram clients. It handles the MTProto protocol, manages connection establishment, and routes requests to appropriate backend services while maintaining persistent connections with clients.

2. **Auth & Session Service** — Manages user authentication through phone number verification, handles session creation and validation, and stores active session tokens. It integrates with SMS providers for sending verification codes.

3. **Message Handling Service** — The core messaging engine that processes incoming and outgoing messages, stores them in the sharded database, manages message queues, and ensures message delivery across multiple devices and data centers.

4. **Media & File Service** — Handles upload and download of media files, including photos, videos, and documents. It manages file streaming, implements chunked uploads for large files, stores file metadata in the database, and interfaces with the distributed file system for actual file storage.

5. **Channel/Broadcast Service** — Manages large-scale broadcasting to channels with unlimited subscribers. It handles message distribution to subscribers, manages channel metadata and permissions, and optimizes message delivery through efficient fan-out mechanisms.

6. **Distributed File System (DFS)** — A custom file storage solution that stores all media files across multiple servers. It provides high availability, redundancy, and fast access to frequently requested files through geographic distribution across Telegram's data centers.

7. **Sharded Chat DB** — A horizontally partitioned database that stores messages, user data, and chat metadata. Sharding is likely based on user or chat IDs to distribute the load evenly and ensure scalability as the user base grows.

## Data flow

![Telegram Sequence Diagram](./out/telegram/sequence-diagram/Sequence%20Diagram.svg)

[Telegram Sequence Diagram code](./src/telegram/sequence-diagram.puml)

### Media Upload and Message Flow:

When a user (Alice) sends a photo to another user (Bob), the following sequence occurs:

1. **Media Upload Phase:** The Mobile App first uploads the photo in encrypted chunks using `saveFilePart` RPC calls. The MTProto Gateway streams this data to the Media Service, which writes it to the Distributed File System (DFS) and stores file metadata in the Sharded DB. Each chunk is acknowledged after successful storage.

2. **Message Send Phase:** Once the file is uploaded, the app sends a `sendMessage` request with a reference to the uploaded file. The Auth Service validates the session, and the Message Service persists the message in both sender's and recipient's message boxes in the database, updates the sequence counter in the cache, and returns a message ID.

3. **Event Propagation:** After message persistence, the Message Service publishes a `NewMessageUpdate` event to the Kafka Event Bus. This event is consumed by the Gateway to update connected clients (showing delivery status) and by the Push Service for offline notification delivery.

4. **Delivery to Recipient:** The Push Service consumes the event, checks Bob's notification settings in the cache and database, and sends a push notification through external providers (FCM/APNs). When Bob opens the app, his client fetches the message update, the photo thumbnail is retrieved from DFS, and the complete message is displayed.

Throughout this flow, components exchange structured data including file chunks, message objects, session tokens, and notification payloads. The separation between synchronous message persistence and asynchronous event propagation ensures fast response times for the sender while guaranteeing eventual delivery to all recipients.

## Deployment

![Telegram Deployment Diagram](./out/telegram/deployment-diagram/Deployment%20Diagram.svg)

[Telegram Deployment Diagram code](./src/telegram/deployment-diagram.puml)

### Deployment Overview:

Telegram's architecture is deployed across a global infrastructure with geographically distributed data centers (DCs). The client tier consists of native mobile apps (iOS/Android), desktop applications, and web clients running in browsers. These clients connect to the Edge/Connection Layer via MTProto protocol over TCP or WebSocket connections.

The Edge Layer contains MTProto Gateways that handle client connections and Bot API Frontends for third-party bot integrations via HTTPS. Behind the edge layer, the Compute Cluster runs containerized microservices (Auth, Message, Channel, Media, and Push services) that communicate via internal RPC using Telegram's TL-Schema protocol.

The data layer is distributed across multiple nodes: an Event Bus Cluster (Kafka or internal implementation) for asynchronous message propagation, an In-Memory Cluster (Redis) for caching session state and sequence counters, and a Storage Cluster containing the Sharded Chat Database and Distributed File System. External integrations include SMS providers for authentication codes and Push Services (FCM/APNs) for mobile notifications.

This deployment architecture enables Telegram to handle millions of concurrent users with low latency by distributing load across multiple data centers and using efficient caching and message routing strategies.

## Assumptions

- I assume the MTProto Gateway implements connection pooling and multiplexing to efficiently handle millions of concurrent client connections with minimal resource overhead.
- I assume the cloud storage system (DFS) implements deduplication to optimize storage costs for shared media files, especially in large group chats and channels where the same file might be sent to multiple recipients.
- I assume the Sharded Chat DB uses consistent hashing or a similar mechanism for data distribution, allowing the system to add new shards dynamically without significant downtime or data migration.
- I assume the Event Bus (Kafka) is configured with appropriate retention policies to handle temporary network partitions and ensure message delivery even when recipients are offline for extended periods.

## Open questions

- How does the data flow look like in secret chats? Specifically, how is end-to-end encryption implemented at the protocol level, and how does the Secret Chat Relay component differ from the standard Message Handling Service?
- What specific load balancing algorithms are used between data centers to route client connections, and how does Telegram handle failover when an entire data center becomes unavailable?
- How does Telegram handle message ordering and consistency across multiple devices when a user has active sessions on their phone, desktop, and web client simultaneously, especially during network instability?
- What caching strategies are implemented in the State Cache (Redis) to optimize read performance for frequently accessed data like recent messages and channel posts, and how is cache invalidation coordinated across the distributed system?
