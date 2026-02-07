## Product Choice

- **Product name:** Telegram
- **Link:** [https://telegram.org](https://telegram.org)
- **Description:** Telegram is a cloud-based instant messaging platform that supports text messages, media sharing, group chats, channels, and bots, with a focus on speed, security, and cross-platform availability.

## Main components

![Telegram Component Diagram](diagrams/out/telegram/component-diagram/Component%20Diagram.svg)

[Telegram Component Diagram Code](diagrams/src/telegram/component-diagram.puml)

### Selected components

1. **MTProto Gateway (DC Entry)** — serves as the entry point for all client connections. It terminates MTProto protocol sessions, routes incoming RPC calls to the appropriate core services, and manages encrypted transport between clients and the backend.

2. **Auth & Session Service** — handles user authentication (phone number verification via SMS), session creation and validation, and manages active device sessions. It interacts with SMS providers for sending verification codes.

3. **Message Handling Service** — processes all incoming and outgoing messages, persists them to the Sharded Chat DB, updates sequence counters in the cache, and publishes new-message events to the Event Bus for asynchronous fan-out.

4. **Media & File Service** — manages upload and download of media files (photos, videos, documents). It streams file data to the Distributed File System (DFS) and stores file metadata in the database.

5. **Notification/Updates Service** — consumes message events from the Event Bus (Kafka) and delivers push notifications to offline users via external push providers (FCM for Android, APNs for iOS). It checks user notification settings before sending.

6. **Sharded Chat DB** — a custom distributed database engine that stores all chat messages, user data, and metadata. It is sharded across multiple nodes to handle Telegram's massive data volume.

## Data flow

![Telegram Sequence Diagram](diagrams/out/telegram/sequence-diagram/Sequence%20Diagram.svg)

[Telegram Sequence Diagram Code](diagrams/src/telegram/sequence-diagram.puml)

### Group: 2. Send Message (with File Ref)

This group describes what happens when Alice sends a message (with an already-uploaded media file reference) to Bob:

1. The **Mobile App** sends an RPC `sendMessage` request (containing the peer ID and file reference) to the **MTProto Gateway**.
2. The **MTProto Gateway** forwards the request to the **Auth Service** to validate Alice's session. The Auth Service confirms the session is valid.
3. The **MTProto Gateway** then passes the message to the **Message Service** for processing.
4. The **Message Service** persists the message (both inbox and outbox copies) to the **Sharded Chat DB** and increments the sequence counter (`pts`) in the **State Cache** (Redis).
5. The **Message Service** returns a confirmation with the assigned message ID to the **MTProto Gateway**, which relays the RPC response back to the **Mobile App**.
6. The app updates Alice's UI with a single tick (message sent).

**Components involved:** Mobile App ↔ MTProto Gateway ↔ Auth Service (session token), MTProto Gateway ↔ Message Service (message payload with file reference), Message Service ↔ Sharded Chat DB (persisted message), Message Service ↔ State Cache (sequence counter).

## Deployment

![Telegram Deployment Diagram](diagrams/out/telegram/deployment-diagram/Deployment%20Diagram.svg)

[Telegram Deployment Diagram Code](diagrams/src/telegram/deployment-diagram.puml)

- **Client Tier:** The Telegram Mobile App runs on user smartphones (iOS/Android). The Desktop App and Web Client (WebA/WebK) run on user computers, with the web client executing inside a browser.
- **Edge / Connection Layer:** The MTProto Gateway and Bot API Frontend are deployed at the edge of Telegram's global infrastructure within the primary data center. They handle all incoming client and bot connections.
- **Compute Cluster:** Core services (Auth, Message, Channel, Media, Push Notification) are deployed as containers/pods within a compute cluster (App Engine) inside the primary DC.
- **Middleware:** Kafka (Event Bus) runs on a dedicated cluster node for asynchronous event processing. Redis (State Cache) runs on an in-memory cluster for fast session and state lookups.
- **Storage Cluster:** The Sharded Chat DB (custom engine) and Distributed File System are deployed on a dedicated storage cluster within the DC.
- **External Ecosystem:** SMS providers (Twilio/others) and push services (FCM/APNs) are external third-party services accessed over the internet.

## Assumptions

- *"I assume the MTProto Gateway performs load balancing across multiple instances of core services within the data center to distribute traffic evenly."*
- *"I assume the State Cache (Redis) is used not only for sequence counters but also for caching recent messages and user online status to reduce database load."*

## Open questions

- *"How does Telegram handle message synchronization and conflict resolution when a user is connected from multiple devices simultaneously?"*
- *"What specific sharding strategy does the custom Chat DB use — is it based on user ID, chat ID, or geographic region?"*
