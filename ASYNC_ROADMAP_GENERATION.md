# Async Roadmap Generation Implementation Guide

## Overview

This document describes the asynchronous roadmap generation system that was implemented to replace the synchronous, blocking approach. Users can now submit roadmap generation requests and poll for progress without blocking the server.

## Architecture

### Backend Flow

```
Frontend (React)
    ↓
POST /ai {scope: ROADMAP, mode: GENERATE}
    ↓
RoadmapScopeHandler
    ├─ Create RoadmapGenerationJob (status=PENDING)
    ├─ Publish to RabbitMQ queue
    └─ Return HTTP 202 with jobId
    ↓
RabbitMQ Queue (roadmap.generation.requests)
    ↓
RoadmapGenerationListener (Consumer)
    ├─ Update job status → PROCESSING
    ├─ Call N8n webhook (with 30s timeout)
    ├─ Exponential backoff retry (3 attempts: 5s, 10s, 20s)
    └─ Update job result or error
    ↓
MongoDB (roadmap_generation_jobs collection)
    └─ Job persists (TTL: 30 days auto-delete)
    ↓
Frontend Polling
    └─ GET /jobs/{jobId} every 1-5 seconds
    └─ Updates progress bar
    └─ Displays results when complete
```

## API Endpoints

### 1. Submit Roadmap Generation (Async)

**Request:**
```bash
POST /ai
Content-Type: application/json

{
  "scope": "ROADMAP",
  "mode": "GENERATE",
  "options": {
    "topic": "React Developer",
    "description": "Learning path from basics to production",
    "maxNodes": 15,
    "roadmapCategoryId": "cat_123"
  }
}
```

**Response (HTTP 202 Accepted):**
```json
{
  "message": "success",
  "result": {
    "data": {
      "jobId": "job_abc123def456",
      "statusUrl": "/jobs/job_abc123def456",
      "createdAt": "2026-05-10T10:00:00Z"
    }
  }
}
```

### 2. Get Job Status (Polling)

**Request:**
```bash
GET /jobs/{jobId}
```

**Response (HTTP 200):**
```json
{
  "message": "success",
  "result": {
    "jobId": "job_abc123def456",
    "status": "PROCESSING",
    "progress": 45,
    "createdAt": "2026-05-10T10:00:00Z",
    "updatedAt": "2026-05-10T10:02:30Z"
  }
}
```

Possible statuses:
- `PENDING` - Waiting to process
- `PROCESSING` - Currently generating
- `COMPLETED` - Done (includes `data` field)
- `FAILED` - Error (includes `errorCode`, `errorMessage`, `retryCount`)

### 3. Get User's Job History

**Request:**
```bash
GET /jobs?limit=20&offset=0&status=COMPLETED
```

**Response:**
```json
{
  "message": "success",
  "result": {
    "jobs": [
      {
        "jobId": "job_xyz",
        "status": "COMPLETED",
        "progress": 100,
        "data": { /* roadmap data */ },
        "createdAt": "2026-05-10T08:00:00Z",
        "completedAt": "2026-05-10T08:05:00Z"
      }
    ],
    "total": 25,
    "limit": 20,
    "offset": 0
  }
}
```

## Frontend Implementation

### Key Components

#### 1. GenerateRoadmapPage
**Location:** `src/pages/roadmap/GenerateRoadmapPage.tsx`

Form to initiate roadmap generation. Features:
- Topic input (required, 1-200 chars)
- Description textarea (optional, 0-1000 chars)
- Max nodes selector (5-50, default 15)
- Difficulty dropdown (BEGINNER, INTERMEDIATE, ADVANCED)
- Form validation
- Submission button with loading state

#### 2. GenerationProgressPage
**Location:** `src/pages/roadmap/GenerationProgressPage.tsx`

Displays job progress. Features:
- Real-time polling with exponential backoff
- 4 status states with different UX:
  - PENDING: Spinner + "Waiting to start..."
  - PROCESSING: Progress bar + elapsed time
  - COMPLETED: Success icon + "View Roadmap" button
  - FAILED: Error icon + error message + "Retry" button
- Manual refresh button
- Smart polling stops when job completes

#### 3. GenerationHistoryPage
**Location:** `src/pages/roadmap/GenerationHistoryPage.tsx`

Shows user's job history. Features:
- Paginated list (10 jobs per page)
- Status filtering
- Action buttons per job:
  - COMPLETED: "View" → roadmap viewer
  - PROCESSING: "View Progress" → progress page
  - FAILED: "Retry" → generation form with prefilled data
- Refresh button
- "Generate New" button

### Key Hooks

#### useGenerationPolling
**Location:** `src/hooks/useGenerationPolling.ts`

Manages polling with exponential backoff:
```typescript
const { job, loading, error, refetch } = useGenerationPolling(jobId);
```

Features:
- Starts at 1s interval, increases to 5s max
- Stops polling when job completes
- Timeout after 30 minutes (180 attempts)
- Cleanup on unmount

#### useGenerationHistory
**Location:** `src/hooks/useGenerationHistory.ts`

Fetches paginated job history:
```typescript
const { jobs, total, loading, loadMore, filterByStatus } = useGenerationHistory(20);
```

Features:
- Pagination
- Status filtering
- Refresh capability

#### useGenerationForm
**Location:** `src/hooks/useGenerationForm.ts`

Form state management:
```typescript
const { form, loading, error, submit, validateForm } = useGenerationForm(onSuccess);
```

Features:
- Form state
- Validation (async submission)
- Error handling

### Services

#### GenerationJobService
**Location:** `src/services/GenerationJobService.ts`

HTTP client for API calls:
```typescript
// Submit generation
const response = await GenerationJobService.submitGeneration({
  topic: 'React',
  description: 'Optional description',
  maxNodes: 15
});

// Get job status
const job = await GenerationJobService.getJobStatus(jobId);

// Get user history
const history = await GenerationJobService.getUserJobs(limit, offset, status);
```

### Redux Integration

**Redux Slice:** `src/store/slices/generationSlice.ts`

State:
```typescript
{
  currentJobId: string | null,
  currentJob: JobStatusResponse | null,
  loading: boolean,
  error: string | null,
  jobHistory: JobStatusResponse[],
  historyLoading: boolean,
  historyError: string | null,
  // ... pagination fields
}
```

Async thunks:
- `submitGenerationAsync` - Submit request
- `fetchJobStatusAsync` - Poll for status
- `fetchHistoryAsync` - Load history

## Data Flow Example

### Scenario: User generates a React learning roadmap

1. **User fills form and submits** (GenerateRoadmapPage)
   - Topic: "React Developer"
   - Max nodes: 15
   - Difficulty: INTERMEDIATE

2. **Frontend submits request**
   ```typescript
   await GenerationJobService.submitGeneration({
     topic: 'React Developer',
     maxNodes: 15
   });
   // Returns jobId: 'job_abc123'
   ```

3. **Backend creates job and queues**
   - RoadmapGenerationJob created with status=PENDING
   - Message published to RabbitMQ

4. **Listener processes job**
   - Status updated to PROCESSING
   - Calls N8n webhook
   - N8n returns roadmap structure
   - Saves result and updates status=COMPLETED

5. **Frontend polling**
   ```typescript
   // useGenerationPolling polls GET /jobs/job_abc123
   // Progress: 0% → 100%
   // Status: PENDING → PROCESSING → COMPLETED
   ```

6. **User sees result**
   - Progress page shows complete roadmap
   - "View Roadmap" button navigates to viewer

## Error Handling

### Retry Logic

If N8n call fails:
- Attempt 1: Immediate
- Attempt 2: After 5 seconds
- Attempt 3: After 10 seconds
- Attempt 4: After 20 seconds

If all fail → Job marked FAILED with error details

### Frontend Error Handling

```typescript
try {
  const response = await GenerationJobService.submitGeneration(request);
} catch (error) {
  toast.error(error.message);
  // Show form validation errors
}
```

Toast notifications show:
- Network errors during polling
- Job submission failures
- Invalid form inputs
- Generation failures

## Performance Optimizations

### Backend
- Job processing offloaded to async queue
- No blocking on N8n calls
- Exponential backoff prevents thundering herd
- MongoDB TTL auto-deletes old jobs

### Frontend
- Polling with exponential backoff (1s → 5s)
- Stops polling when complete
- Redux state reduces re-renders
- Hook memoization prevents unnecessary API calls

## Testing Checklist

### Backend
- [ ] Unit test RoadmapGenerationJobService CRUD
- [ ] Unit test RoadmapGenerationListener retry logic
- [ ] Integration: submit → queue → process → complete
- [ ] Integration: timeout → retry succeeds
- [ ] Integration: all retries fail → FAILED status
- [ ] User can only access own jobs (403 forbidden)
- [ ] Load test: 10 concurrent requests

### Frontend
- [ ] Form validation works
- [ ] Submit navigates to progress page
- [ ] Polling updates progress bar
- [ ] PENDING → PROCESSING → COMPLETED flow
- [ ] FAILED status shows error and retry button
- [ ] History page paginates
- [ ] Status filtering works
- [ ] Mobile responsive (375px width)

## Deployment Notes

### Prerequisites
- RabbitMQ running (docker-compose)
- MongoDB running (docker-compose)
- N8n webhook configured and accessible

### Environment Variables (Backend)
```env
N8N_GEN_ROADMAP_URL=http://n8n:5678/webhook/roadmap
RABBITMQ_HOST=rabbitmq
RABBITMQ_PORT=5672
RABBITMQ_USERNAME=guest
RABBITMQ_PASSWORD=guest
MONGODB_URL=mongodb://localhost:27017/roadmap_ai
```

### Steps
1. Deploy backend service
2. Verify MongoDB indexes are created
3. Verify RabbitMQ queue is created
4. Deploy frontend
5. Test E2E flow

## Monitoring

### Key Metrics
- Job processing latency (target: < 5 min avg)
- Error rate (target: < 1%)
- Queue depth (target: 0-5 at steady state)
- Polling request rate (target: < 1000 req/sec)

### Logs to Watch
- RoadmapGenerationListener: processing, retry attempts, errors
- N8nServiceImpl: N8n calls, timeouts
- JobStatusController: access control, not found errors

## Future Enhancements

1. **WebSocket real-time updates** instead of polling
2. **Job cancellation** mid-process
3. **Progress estimation** using ML
4. **Email notification** when complete
5. **Batch generation** for multiple roadmaps
6. **Job sharing** with other users
7. **Template library** for common topics
