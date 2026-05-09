# Hocuspocus Server

## Overview

The Hocuspocus Server provides real-time collaborative editing for roadmaps using the Y.js CRDT protocol. It loads roadmap data from MongoDB into shared Y.js documents, syncs changes between connected clients, and persists updates back to the roadmap service via a Redis Stream.

## Port

| Protocol  | Port |
| --------- | ---- |
| WebSocket | 1234 |

## Technology Stack

- **Node.js** + **TypeScript**
- **@hocuspocus/server** — Y.js CRDT WebSocket server
- **MongoDB** (Mongoose/native driver) — source of truth for roadmap data
- **Redis** (ioredis) — Redis Streams for publishing change events

## How It Works

### Document Load (`onLoadDocument`)

When a client connects to edit a roadmap, the server:

1. Resolves the roadmap ID from the document name
2. Queries MongoDB (`roadmap_service.RoadMap`) with a `$lookup` on `RoadMapCategory`
3. Populates the shared Y.js document with four maps:
   - `nodes` — roadmap nodes keyed by `nodeId`
   - `edges` — roadmap edges keyed by `edgeId`
   - `meta` — roadmap name
   - `roadmapInfo` — full roadmap metadata (name, description, category)

### Document Store (`onStoreDocument`)

When clients stop editing (idle save), the server:

1. Extracts current state from the Y.js document maps
2. Publishes a `DOCUMENT_UPDATED` event to the Redis Stream `roadmap-document-updates`

The Roadmap Service consumes this stream to persist changes back to MongoDB.

## Redis Stream Schema

Stream key: `roadmap-document-updates`

| Field          | Type   | Description                     |
| -------------- | ------ | ------------------------------- |
| `eventType`    | string | Always `DOCUMENT_UPDATED`       |
| `roadmapId`    | string | MongoDB roadmap document ID     |
| `nodes`        | JSON   | Array of node objects           |
| `edges`        | JSON   | Array of edge objects           |
| `roadmapInfo`  | JSON   | `{name, description, category}` |
| `timestamp`    | string | ISO 8601 timestamp              |

## Environment Variables

| Variable            | Default                                                        | Description                          |
| ------------------- | -------------------------------------------------------------- | ------------------------------------ |
| `HOCUSPOCUS_PORT`   | `1234`                                                         | WebSocket server port                |
| `MONGODB_URL`       | `mongodb://roadmap_user:roadmap_pass123@<host>:27017?...`      | MongoDB connection string            |
| `REDIS_HOST`        | `localhost`                                                    | Redis host                           |
| `REDIS_PORT`        | `6382`                                                         | Redis port                           |
| `REDIS_STREAM_KEY`  | `roadmap-document-updates`                                     | Redis Stream key for change events   |

## Dependencies

| Dependency        | Role                                             |
| ----------------- | ------------------------------------------------ |
| MongoDB           | Load initial roadmap state on document open      |
| Redis             | Publish `DOCUMENT_UPDATED` events to stream      |
| Roadmap Service   | Consumes Redis stream to persist changes to DB   |
| memap-frontend    | Connects via WebSocket for collaborative editing |

## Running Locally

```bash
cd hocuspocus-server
yarn install
yarn dev
```

Or via Docker Compose:

```bash
docker-compose up hocuspocus-server
```
