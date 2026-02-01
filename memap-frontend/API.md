# MeMap Frontend API Integration Guide

## Overview

This document describes how the frontend integrates with backend services through APIs. All API calls go through an API Gateway at port 8280.

## API Base URLs

```typescript
// Development
const API_GATEWAY = 'http://localhost:8280';

// Services
const SERVICES = {
  identity: `${API_GATEWAY}/identity-service/identity`,
  profile: `${API_GATEWAY}/profile-service/profile/api`,
  roadmap: `${API_GATEWAY}/roadmap-service/road-map/api`,
  file: `${API_GATEWAY}/file-service/file`,
  ai: `${API_GATEWAY}/roadmap-ai-service`,
  learning: `${API_GATEWAY}/learning-service/learning/api/v1`,
};
```

## Authentication

### Login Flow

```typescript
// 1. User clicks login
// 2. Redirect to Keycloak
window.location.href = `${KEYCLOAK_URL}/realms/${REALM}/protocol/openid-connect/auth?client_id=${CLIENT_ID}&redirect_uri=${REDIRECT_URI}&response_type=code`;

// 3. After successful login, Keycloak redirects back with code
// 4. Exchange code for tokens
const response = await axios.post(
  `${KEYCLOAK_URL}/realms/${REALM}/protocol/openid-connect/token`,
  {
    grant_type: 'authorization_code',
    client_id: CLIENT_ID,
    code: authCode,
    redirect_uri: REDIRECT_URI,
  },
);

// 5. Store tokens
localStorage.setItem('accessToken', response.data.access_token);
localStorage.setItem('refreshToken', response.data.refresh_token);

// 6. Decode token to get user info
const decoded = jwtDecode(response.data.access_token);
dispatch(setUser(decoded));
```

### Token Refresh

```typescript
// Automatic token refresh with interceptor
axios.interceptors.response.use(
  (response) => response,
  async (error) => {
    if (error.response?.status === 401) {
      const refreshToken = localStorage.getItem('refreshToken');

      try {
        const response = await axios.post(
          `${KEYCLOAK_URL}/realms/${REALM}/protocol/openid-connect/token`,
          {
            grant_type: 'refresh_token',
            client_id: CLIENT_ID,
            refresh_token: refreshToken,
          },
        );

        localStorage.setItem('accessToken', response.data.access_token);

        // Retry original request
        error.config.headers['Authorization'] =
          `Bearer ${response.data.access_token}`;
        return axios(error.config);
      } catch (refreshError) {
        // Refresh failed, logout
        dispatch(logout());
        return Promise.reject(refreshError);
      }
    }
    return Promise.reject(error);
  },
);
```

## RTK Query Service Definitions

### Roadmap Service

```typescript
// src/services/api/roadmapApi.ts
export const roadmapApi = createApi({
  reducerPath: 'roadmapApi',
  baseQuery: fetchBaseQuery({
    baseUrl: SERVICES.roadmap,
    prepareHeaders: (headers, { getState }) => {
      const token = selectAuthToken(getState());
      if (token) {
        headers.set('authorization', `Bearer ${token}`);
      }
      return headers;
    },
  }),
  tagTypes: ['Roadmap', 'Node'],
  endpoints: (builder) => ({
    // Get roadmap by ID
    getRoadmap: builder.query<RoadmapResponse, string>({
      query: (id) => `/roadmap/${id}`,
      providesTags: (result, error, id) => [{ type: 'Roadmap', id }],
    }),

    // Get user's roadmaps
    getMyRoadmaps: builder.query<
      PageResponse<RoadmapResponse>,
      GetMyRoadmapsParams
    >({
      query: ({ page, size, scope, search }) => ({
        url: '/roadmap/my-roadmap',
        params: { page, size, scope, search },
      }),
      providesTags: ['Roadmap'],
    }),

    // Create roadmap
    createRoadmap: builder.mutation<
      CreateRoadmapResponse,
      CreateRoadmapRequest
    >({
      query: (body) => ({
        url: '/roadmap',
        method: 'POST',
        body,
      }),
      invalidatesTags: ['Roadmap'],
    }),

    // Update node
    updateNode: builder.mutation<boolean, UpdateNodeParams>({
      query: ({ roadmapId, nodeRequest }) => ({
        url: `/roadmap/node/${roadmapId}`,
        method: 'PATCH',
        body: nodeRequest,
      }),
      invalidatesTags: (result, error, { roadmapId }) => [
        { type: 'Roadmap', id: roadmapId },
        { type: 'Node' },
      ],
    }),

    // Update edge
    updateEdge: builder.mutation<boolean, UpdateEdgeParams>({
      query: ({ roadmapId, edgeRequest }) => ({
        url: `/roadmap/edge/${roadmapId}`,
        method: 'PATCH',
        body: edgeRequest,
      }),
      invalidatesTags: (result, error, { roadmapId }) => [
        { type: 'Roadmap', id: roadmapId },
      ],
    }),

    // Upload roadmap image
    uploadImage: builder.mutation<FileResponse, UploadImageParams>({
      query: ({ roadmapId, file }) => {
        const formData = new FormData();
        formData.append('file', file);

        return {
          url: `/roadmap/image/${roadmapId}`,
          method: 'PATCH',
          body: formData,
        };
      },
      invalidatesTags: (result, error, { roadmapId }) => [
        { type: 'Roadmap', id: roadmapId },
      ],
    }),

    // Get progress tracking
    getProgress: builder.query<RoadmapTrackingProgressResponse, string>({
      query: (roadmapId) => `/roadmap-tracking-progress/${roadmapId}`,
    }),

    // Update node tracking
    updateNodeTracking: builder.mutation<boolean, UpdateNodeTrackingParams>({
      query: ({ roadmapId, request }) => ({
        url: `/roadmap-tracking-progress/${roadmapId}`,
        method: 'PATCH',
        body: request,
      }),
    }),
  }),
});

export const {
  useGetRoadmapQuery,
  useGetMyRoadmapsQuery,
  useCreateRoadmapMutation,
  useUpdateNodeMutation,
  useUpdateEdgeMutation,
  useUploadImageMutation,
  useGetProgressQuery,
  useUpdateNodeTrackingMutation,
} = roadmapApi;
```

### AI Service

```typescript
// src/services/api/aiApi.ts
export const aiApi = createApi({
  reducerPath: 'aiApi',
  baseQuery: fetchBaseQuery({ baseUrl: SERVICES.ai }),
  endpoints: (builder) => ({
    // Generate roadmap
    generateRoadmap: builder.mutation<
      GenerateRoadmapResponse,
      GenerateRoadmapRequest
    >({
      query: (body) => ({
        url: '/roadmap/generate',
        method: 'POST',
        body,
      }),
    }),

    // Summarize node
    summarizeNode: builder.mutation<NodeSummaryResponse, NodeSummaryRequest>({
      query: (body) => ({
        url: '/node/summary',
        method: 'POST',
        body,
      }),
    }),

    // Ask question about node
    askNodeQuestion: builder.mutation<ChatNodeResponse, ChatNodeRequest>({
      query: (body) => ({
        url: '/node/ask',
        method: 'POST',
        body,
      }),
    }),

    // Generate quiz
    generateQuiz: builder.mutation<QuestionListResponseDto, ChatRoadmapRequest>(
      {
        query: (body) => ({
          url: '/chat',
          method: 'POST',
          body,
        }),
      },
    ),
  }),
});

export const {
  useGenerateRoadmapMutation,
  useSummarizeNodeMutation,
  useAskNodeQuestionMutation,
  useGenerateQuizMutation,
} = aiApi;
```

### Quiz Service

```typescript
// src/services/api/quizApi.ts
export const quizApi = createApi({
  reducerPath: 'quizApi',
  baseQuery: fetchBaseQuery({
    baseUrl: SERVICES.learning,
    prepareHeaders: (headers, { getState }) => {
      const token = selectAuthToken(getState());
      if (token) {
        headers.set('authorization', `Bearer ${token}`);
      }
      return headers;
    },
  }),
  tagTypes: ['Quiz'],
  endpoints: (builder) => ({
    // Create quiz
    createQuiz: builder.mutation<QuizSummaryResponse, CreateQuizRequest>({
      query: (body) => ({
        url: '/quizzes',
        method: 'POST',
        body,
      }),
      invalidatesTags: ['Quiz'],
    }),

    // Get quiz by ID
    getQuiz: builder.query<QuizDetailResponse, string>({
      query: (id) => `/quizzes/${id}`,
      providesTags: (result, error, id) => [{ type: 'Quiz', id }],
    }),

    // Get my quizzes
    getMyQuizzes: builder.query<QuizListResponse, GetMyQuizzesParams>({
      query: (params) => ({
        url: '/quizzes/my-quizzes',
        params,
      }),
      providesTags: ['Quiz'],
    }),

    // Toggle visibility
    toggleVisibility: builder.mutation<
      QuizSummaryResponse,
      ToggleVisibilityParams
    >({
      query: ({ quizId, visible }) => ({
        url: `/quizzes/${quizId}/visibility`,
        method: 'PUT',
        body: { visible },
      }),
      invalidatesTags: (result, error, { quizId }) => [
        { type: 'Quiz', id: quizId },
        'Quiz',
      ],
    }),
  }),
});

export const {
  useCreateQuizMutation,
  useGetQuizQuery,
  useGetMyQuizzesQuery,
  useToggleVisibilityMutation,
} = quizApi;
```

## Component Usage Examples

### Fetching Data with RTK Query

```tsx
import { useGetRoadmapQuery } from '../services/api/roadmapApi';

function ViewRoadmapPage() {
  const { id } = useParams<{ id: string }>();
  const { data, isLoading, error } = useGetRoadmapQuery(id!);

  if (isLoading) return <Loading />;
  if (error) return <Error message="Failed to load roadmap" />;

  return (
    <div>
      <h1>{data?.title}</h1>
      <RoadmapViewer nodes={data?.nodes} edges={data?.edges} />
    </div>
  );
}
```

### Creating Data with Mutations

```tsx
import { useCreateRoadmapMutation } from '../services/api/roadmapApi';

function CreateRoadmapForm() {
  const [createRoadmap, { isLoading }] = useCreateRoadmapMutation();

  const handleSubmit = async (formData: CreateRoadmapRequest) => {
    try {
      const result = await createRoadmap(formData).unwrap();
      navigate(`/roadmap/${result.roadmapId}`);
    } catch (error) {
      toast.error('Failed to create roadmap');
    }
  };

  return (
    <form onSubmit={handleSubmit}>
      {/* Form fields */}
      <button type="submit" disabled={isLoading}>
        {isLoading ? 'Creating...' : 'Create Roadmap'}
      </button>
    </form>
  );
}
```

### File Upload

```tsx
import { useUploadImageMutation } from '../services/api/roadmapApi';

function RoadmapImageUpload({ roadmapId }: { roadmapId: string }) {
  const [uploadImage, { isLoading }] = useUploadImageMutation();

  const handleFileChange = async (e: React.ChangeEvent<HTMLInputElement>) => {
    const file = e.target.files?.[0];
    if (!file) return;

    try {
      await uploadImage({ roadmapId, file }).unwrap();
      toast.success('Image uploaded successfully');
    } catch (error) {
      toast.error('Failed to upload image');
    }
  };

  return (
    <input
      type="file"
      accept="image/*"
      onChange={handleFileChange}
      disabled={isLoading}
    />
  );
}
```

## Error Handling

### Global Error Handler

```typescript
// src/utils/errorHandler.ts
export const handleApiError = (error: any) => {
  if (error.status === 401) {
    // Unauthorized - redirect to login
    window.location.href = '/auth';
  } else if (error.status === 403) {
    // Forbidden - show error message
    toast.error("You don't have permission to perform this action");
  } else if (error.status === 404) {
    // Not found
    toast.error('Resource not found');
  } else if (error.data?.message) {
    // Server error with message
    toast.error(error.data.message);
  } else {
    // Generic error
    toast.error('An error occurred. Please try again.');
  }
};
```

### Using Error Handler

```tsx
import { handleApiError } from '../utils/errorHandler';

function MyComponent() {
  const [updateNode, { isLoading }] = useUpdateNodeMutation();

  const handleUpdate = async (data: NodeRequest) => {
    try {
      await updateNode({ roadmapId, nodeRequest: data }).unwrap();
      toast.success('Node updated successfully');
    } catch (error) {
      handleApiError(error);
    }
  };

  return <NodeForm onSubmit={handleUpdate} />;
}
```

## Type Definitions

### Common Types

```typescript
// src/types/api.ts
export interface ApiResponse<T> {
  code: number;
  message?: string;
  result: T;
}

export interface PageResponse<T> {
  data: T[];
  currentPage: number;
  totalPages: number;
  pageSize: number;
  totalElements: number;
}

// Roadmap Types
export interface RoadmapResponse {
  roadmapId: string;
  title: string;
  description: string;
  scope: 'PRIVATE' | 'GROUP' | 'PUBLIC';
  ownerId: string;
  categoryId: string;
  imageUrl?: string;
  nodes: Node[];
  edges: Edge[];
  createdAt: string;
  updatedAt: string;
}

export interface Node {
  nodeId: string;
  label: string;
  type: string;
  position: { x: number; y: number };
  data: {
    description: string;
    resources: Resource[];
  };
}

export interface Edge {
  edgeId: string;
  source: string;
  target: string;
  type: string;
  label?: string;
}

// Quiz Types
export interface QuizDetailResponse {
  quizId: string;
  title: string;
  description: string;
  roadmapId: string;
  visible: boolean;
  teacherId: string;
  questions: Question[];
  createdAt: string;
  updatedAt: string;
}

export interface Question {
  questionId: string;
  questionText: string;
  options: Option[];
}

export interface Option {
  optionId: string;
  optionText: string;
  isCorrect: boolean;
}
```

## API Rate Limiting

The frontend implements client-side rate limiting to prevent abuse:

```typescript
// src/utils/rateLimiter.ts
class RateLimiter {
  private requests: Map<string, number[]> = new Map();

  canMakeRequest(key: string, limit: number, windowMs: number): boolean {
    const now = Date.now();
    const timestamps = this.requests.get(key) || [];

    // Remove old timestamps outside the window
    const validTimestamps = timestamps.filter((ts) => now - ts < windowMs);

    if (validTimestamps.length >= limit) {
      return false;
    }

    validTimestamps.push(now);
    this.requests.set(key, validTimestamps);
    return true;
  }
}

export const rateLimiter = new RateLimiter();

// Usage in component
const handleAIGeneration = () => {
  if (!rateLimiter.canMakeRequest('ai-generate', 5, 60000)) {
    toast.error('Too many requests. Please wait a minute.');
    return;
  }

  // Proceed with AI generation
  generateRoadmap(requestData);
};
```

## WebSocket Integration (Future)

For real-time collaboration:

```typescript
// src/services/websocket.ts
import { io, Socket } from 'socket.io-client';

class WebSocketService {
  private socket: Socket | null = null;

  connect(roadmapId: string, token: string) {
    this.socket = io('ws://localhost:8083/road-map/ws', {
      auth: { token },
      query: { roadmapId },
    });

    this.socket.on('node-updated', (data) => {
      // Handle node update from other users
      dispatch(updateNodeFromSocket(data));
    });

    this.socket.on('edge-updated', (data) => {
      // Handle edge update
      dispatch(updateEdgeFromSocket(data));
    });
  }

  disconnect() {
    this.socket?.disconnect();
  }

  emitNodeUpdate(nodeData: Node) {
    this.socket?.emit('update-node', nodeData);
  }
}

export const wsService = new WebSocketService();
```

## Best Practices

1. **Use RTK Query**: Leverage caching and automatic refetching
2. **Type Safety**: Always define TypeScript types for API responses
3. **Error Handling**: Implement consistent error handling across the app
4. **Loading States**: Show loading indicators for better UX
5. **Optimistic Updates**: Update UI immediately, rollback on error
6. **Token Management**: Automatic refresh, secure storage
7. **Rate Limiting**: Prevent API abuse with client-side limits
8. **Retry Logic**: Implement retry for failed requests
9. **Request Cancellation**: Cancel ongoing requests when components unmount
10. **Data Normalization**: Normalize nested data for easier state management
