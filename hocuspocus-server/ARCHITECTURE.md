# Hocuspocus Server Architecture

## Role in the System

The Hocuspocus Server is the real-time collaborative editing backend for roadmaps. It bridges the Y.js CRDT protocol used in the browser with the MongoDB data store and Redis change-event pipeline used by the Roadmap Service.

```
                Browser Clients (React + Y.js)
                         │
                         │ WebSocket :1234 (Y.js/Hocuspocus protocol)
                         ▼
           ┌─────────────────────────┐
           │    Hocuspocus Server    │
           │    (Node.js / TS)       │
           │                         │
           │  onLoadDocument ──────► MongoDB
           │  (reads roadmap state)  │ (roadmap_service.RoadMap)
           │                         │
           │  onStoreDocument ─────► Redis Stream
           │  (publishes changes)    │ (roadmap-document-updates)
           └─────────────────────────┘
                                     │
                                     ▼
                           Roadmap Service
                           (consumes stream,
                            persists to MongoDB)
```

## Data Flow: Collaborative Editing Session

```
1. User opens roadmap editor in browser
   └─► WebSocket connect to ws://hocuspocus-server:1234/{roadmapId}

2. onLoadDocument fires
   ├─► Query MongoDB: RoadMap + $lookup RoadMapCategory
   └─► Populate Y.Doc maps:
       ├─ nodesMap  (key: nodeId  → node object)
       ├─ edgesMap  (key: edgeId  → edge object)
       ├─ metaMap   (key: "name"  → roadmap name)
       └─ roadmapInfo (key: "roadmapInfo" → {name, desc, category})

3. Users edit collaboratively
   └─► Y.js CRDTs merge changes; server relays diffs to all peers

4. Document saved (idle / disconnect)
   └─► onStoreDocument fires
       ├─► Extract nodes[], edges[], roadmapInfo from Y.Doc
       └─► XADD roadmap-document-updates * eventType DOCUMENT_UPDATED
               roadmapId <id> nodes <json> edges <json>
               roadmapInfo <json> timestamp <iso>

5. Roadmap Service (stream consumer)
   └─► Reads DOCUMENT_UPDATED from Redis Stream
       └─► Persists updated nodes/edges to MongoDB
```

## Y.js Document Structure

Each document (named by roadmap ID) has four shared maps:

| Map name       | Key           | Value type     | Description                      |
| -------------- | ------------- | -------------- | -------------------------------- |
| `nodes`        | `nodeId`      | node object    | All roadmap nodes                |
| `edges`        | `edgeId`      | edge object    | All roadmap edges                |
| `meta`         | `"name"`      | string         | Roadmap display name             |
| `roadmapInfo`  | `"roadmapInfo"`| object        | `{name, description, category}`  |

## MongoDB ID Resolution

The server handles both string and `ObjectId` `_id` types, since Spring Data MongoDB may store IDs differently depending on the field type annotation:

1. Try `findOne({ _id: documentName })` (string)
2. Fall back to `findOne({ _id: new ObjectId(documentName) })`

## Fault Tolerance

- **MongoDB**: Connection failure is logged but does not crash the server. The Y.Doc remains the source of truth in memory.
- **Redis**: Reconnects automatically with exponential backoff (500 ms × attempt, max 5 s). If `XADD` fails during `onStoreDocument`, the error is logged and swallowed — the next `onStoreDocument` call will publish the latest state.

## Integration Points

| System          | Direction     | Protocol        | Purpose                                 |
| --------------- | ------------- | --------------- | --------------------------------------- |
| Browser         | Bidirectional | WebSocket (Y.js)| CRDT sync for collaborative editing     |
| MongoDB         | Read          | MongoDB driver  | Load initial roadmap state              |
| Redis           | Write         | Redis Streams   | Publish `DOCUMENT_UPDATED` events       |
| Roadmap Service | Downstream    | Redis Streams   | Consumes events to persist to database  |
