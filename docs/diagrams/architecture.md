# Telegram Architecture Documentation

## Overview

- **Product**: Telegram Messenger
- **Official Website**: [telegram.org](https://telegram.org)
- **Description**: A cloud-based instant messaging service with a focus on speed, security, and cross-platform synchronization.

This document provides an architectural overview of Telegram's system design, including component structure, deployment topology, and message flow patterns.

---

## Component Architecture

The following diagram illustrates the high-level component architecture of the Telegram ecosystem:

![Telegram Component Diagram](./out/telegram/component-diagram/Component%20Diagram.svg)

[View PlantUML source](./src/telegram/component-diagram.puml)

### Key Components

#### Client Applications
- **Mobile Apps (iOS/Android)**: Native mobile clients for smartphones and tablets
- **Desktop App**: Cross-platform desktop application for Windows, macOS, and Linux
- **Web Client (K/Z)**: Browser-based clients (WebK and WebZ) accessible via web.telegram.org

#### Connection Layer
- **MTProto Gateway**: Entry point for client connections using Telegram's proprietary MTProto protocol. Handles connection management, encryption, and load balancing across data centers.
- **Bot API Server**: REST API endpoint for third-party bots, translating HTTP requests to internal protocol.

#### Core Services
- **Auth & Session Service**: Manages user authentication via phone numbers (SMS verification), session token management, and 2FA.
- **Message Handling Service**: Central service for processing text messages, coordinating delivery, and maintaining message sequences.
- **Channel/Broadcast Service**: Handles large-scale broadcasting for public channels and group messaging.
- **Media & File Service**: Processes file uploads/downloads, manages media storage, and handles file deduplication.
- **Notification/Updates Service**: Manages push notifications and real-time updates to connected clients.
- **Secret Chat Relay**: Facilitates end-to-end encrypted secret chats with client-side encryption.

#### Data Layer
- **Sharded Chat DB**: Horizontally sharded database storing chat history, user data, and message metadata.
- **Distributed File System (DFS)**: Stores media files with geographic distribution and redundancy.
- **State Cache (Redis)**: In-memory cache for session states, user presence, and frequently accessed data.

#### Event Processing
- **Event Bus (Kafka/Internal)**: Asynchronous message queue for decoupling services and enabling real-time updates propagation.

#### External Integrations
- **SMS Providers**: Third-party services (e.g., Twilio) for authentication code delivery.
- **Push Services**: FCM (Firebase Cloud Messaging) for Android and APNs (Apple Push Notification service) for iOS.

---

## Deployment Architecture

The following diagram shows the physical deployment topology of Telegram's infrastructure:

![Telegram Deployment Diagram](./out/telegram/deployment-diagram/Deployment%20Diagram.svg)

[View PlantUML source](./src/telegram/deployment-diagram.puml)

### Deployment Topology

#### Client Tier
- **User Devices**: Smartphones, computers, and web browsers running Telegram client applications.
- **Third-Party Servers**: External bot servers operated by developers integrating with Telegram's Bot API.

#### Edge/Connection Layer
Geographically distributed gateway nodes handling:
- MTProto protocol termination for client connections
- TLS/SSL offloading for bot HTTPS requests
- DDoS protection and traffic filtering
- Connection pooling and load balancing

#### Application Tier (Compute Cluster)
- Containerized microservices running on orchestration platforms (Kubernetes/custom)
- Horizontal scaling based on load metrics
- Service mesh for inter-service communication
- Health checks and automatic failover

#### Data Tier
- **Sharded Database**: Custom storage engine optimized for Telegram's access patterns, distributed across multiple nodes for horizontal scaling
- **Distributed File System**: Media files stored with replication and geographic distribution for low-latency access
- **Cache Cluster**: Redis cluster for session management and hot data caching

#### Middleware
- **Event Bus**: Kafka or custom message broker for asynchronous communication between services
- Enables event-driven architecture for real-time features

#### External Ecosystem
- Integration with third-party services for SMS delivery and push notifications
- API gateways for bot developers

### Infrastructure Characteristics

- **Geographic Distribution**: Multiple data centers worldwide for reduced latency
- **High Availability**: Redundant components with automatic failover mechanisms
- **Scalability**: Horizontal scaling at every tier to handle millions of concurrent users
- **Security**: End-to-end encryption for secret chats, encrypted storage, and secure inter-service communication

---

## Message Flow (Sequence Diagram)

For a detailed view of how a media message flows through the system, see the sequence diagram:

![Telegram Sequence Diagram](./out/telegram/sequence-diagram/Sequence%20Diagram.svg)

[View PlantUML source](./src/telegram/sequence-diagram.puml)

### Media Message Flow Overview

1. **Media Upload Phase**: Client uploads file in encrypted chunks before sending the message
2. **Message Send Phase**: Message with file reference is submitted and persisted
3. **Event Propagation**: Asynchronous fan-out of message delivery events
4. **Recipient Delivery**: Push notification triggers recipient's app to fetch and display the message

---

## Technical Considerations

### Data Deduplication
The distributed file system implements content-addressable storage with deduplication to optimize storage costs for shared media files (e.g., frequently forwarded photos or videos).

### Secret Chats
Secret chats use end-to-end encryption with keys generated on client devices. The server acts as a relay without access to message content. Messages are not stored on servers and are available only on the participating devices.

### MTProto Protocol
Telegram's custom MTProto protocol is optimized for mobile networks, supporting:
- Multiple data centers with automatic DC migration
- Connection recovery after network interruptions
- Efficient binary serialization (TL schema)
- Built-in encryption layer

### Scalability Strategy
- Database sharding by user ID ranges
- Geographic distribution of data and compute resources
- CDN-like architecture for media delivery
- Aggressive caching at multiple levels
