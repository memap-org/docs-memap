# MeMap Frontend Architecture

## Architecture Overview

The MeMap frontend is a single-page application (SPA) built with React 18.3, TypeScript 5.6, and Vite 6.0. It follows a modular, component-based architecture with clear separation of concerns.

```
┌─────────────────────────────────────────────────────────────┐
│                        Browser                               │
│  ┌───────────────────────────────────────────────────────┐  │
│  │              React Application (SPA)                   │  │
│  │  ┌─────────────────────────────────────────────────┐  │  │
│  │  │         Presentation Layer (Components)         │  │  │
│  │  │  - Pages, Layouts, UI Components                │  │  │
│  │  │  - Tailwind CSS for styling                     │  │  │
│  │  └─────────────────────────────────────────────────┘  │  │
│  │  ┌─────────────────────────────────────────────────┐  │  │
│  │  │         State Management (Redux Toolkit)        │  │  │
│  │  │  - Auth, Roadmap, Quiz, UI slices               │  │  │
│  │  │  - RTK Query for API caching                    │  │  │
│  │  └─────────────────────────────────────────────────┘  │  │
│  │  ┌─────────────────────────────────────────────────┐  │  │
│  │  │         Routing Layer (React Router v6)         │  │  │
│  │  │  - Public, Protected, Teacher routes            │  │  │
│  │  │  - Route guards with role checking              │  │  │
│  │  └─────────────────────────────────────────────────┘  │  │
│  │  ┌─────────────────────────────────────────────────┐  │  │
│  │  │         Service Layer (API Integration)         │  │  │
│  │  │  - Axios HTTP client                            │  │  │
│  │  │  - RTK Query API services                       │  │  │
│  │  │  - Token management & refresh                   │  │  │
│  │  └─────────────────────────────────────────────────┘  │  │
│  └───────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
                            │
                            │ HTTPS/WSS
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                    API Gateway (Port 8280)                   │
│  Routes to: Identity, Profile, Roadmap, File, AI, Learning  │
└─────────────────────────────────────────────────────────────┘
                            │
          ┌─────────────────┼─────────────────┐
          ▼                 ▼                 ▼
┌─────────────────┐ ┌──────────────┐ ┌──────────────┐
│ Identity Service│ │Roadmap Service│ │ AI Service   │
│   (Keycloak)    │ │  (Port 8083)  │ │ (Port 8084)  │
└─────────────────┘ └──────────────┘ └──────────────┘
          ▼                 ▼                 ▼
┌─────────────────┐ ┌──────────────┐ ┌──────────────┐
│ Profile Service │ │ File Service │ │Learning Srvc │
│  (Port 8082)    │ │  (Port 8085) │ │ (Port 8086)  │
└─────────────────┘ └──────────────┘ └──────────────┘
```

## Project Structure

### Directory Organization

```
src/
├── assets/                  # Static assets (images, fonts, icons)
│   ├── icons/              # SVG icons
│   ├── images/             # Images (logos, backgrounds)
│   └── fonts/              # Custom fonts
│
├── components/              # Reusable UI components
│   ├── common/             # Generic components (Button, Modal, Input)
│   ├── layout/             # Layout components (Header, Footer, Sidebar)
│   ├── roadmap/            # Roadmap-specific components
│   │   ├── RoadmapViewer.tsx
│   │   ├── RoadmapEditor.tsx
│   │   ├── NodeEditor.tsx
│   │   └── EdgeEditor.tsx
│   ├── quiz/               # Quiz components
│   │   ├── QuizBuilder.tsx
│   │   ├── QuizPlayer.tsx
│   │   └── QuestionCard.tsx
│   └── auth/               # Auth components (LoginForm, ProtectedRoute)
│
├── pages/                   # Page-level components
│   ├── HomePage.tsx
│   ├── Dashboard.tsx
│   ├── ViewRoadmapPage.tsx
│   ├── EditorRoadmapPage.tsx
│   ├── MultiChoiceQuizPage.tsx
│   ├── Learning.tsx
│   └── Account.tsx
│
├── routes/                  # Routing configuration
│   ├── AppRoute.tsx        # Main route definitions
│   └── ProtectedRoute.tsx  # Route guard for auth/roles
│
├── store/                   # Redux store configuration
│   ├── index.ts            # Store setup
│   ├── slices/             # Redux slices
│   │   ├── authSlice.ts
│   │   ├── roadmapSlice.ts
│   │   ├── quizSlice.ts
│   │   └── uiSlice.ts
│   └── middleware/         # Custom middleware
│       └── authMiddleware.ts
│
├── services/                # API services
│   ├── api/                # RTK Query API definitions
│   │   ├── roadmapApi.ts
│   │   ├── aiApi.ts
│   │   ├── quizApi.ts
│   │   ├── profileApi.ts
│   │   └── fileApi.ts
│   ├── http/               # Axios configuration
│   │   ├── axiosClient.ts
│   │   └── interceptors.ts
│   └── websocket/          # WebSocket service (future)
│       └── wsService.ts
│
├── hooks/                   # Custom React hooks
│   ├── useAuth.ts
│   ├── useRoadmap.ts
│   ├── useDebounce.ts
│   └── useLocalStorage.ts
│
├── utils/                   # Utility functions
│   ├── validation.ts       # Form validation
│   ├── formatting.ts       # Date, string formatting
│   ├── errorHandler.ts     # Error handling
│   ├── tokenManager.ts     # JWT token management
│   └── rateLimiter.ts      # Client-side rate limiting
│
├── types/                   # TypeScript type definitions
│   ├── api.ts              # API response types
│   ├── models.ts           # Domain models
│   ├── auth.ts             # Auth-related types
│   └── components.ts       # Component prop types
│
├── constants/               # Application constants
│   ├── routes.ts           # Route paths
│   ├── api.ts              # API endpoints
│   └── config.ts           # App configuration
│
├── styles/                  # Global styles
│   ├── index.css           # Tailwind imports
│   └── themes/             # Theme configurations
│
├── App.tsx                  # Root component
├── main.tsx                 # Entry point
└── vite-env.d.ts           # Vite type definitions
```

## Core Architectural Layers

### 1. Presentation Layer

#### Component Architecture

```
Component Hierarchy:
App
├── AppRoute (Router)
│   ├── PublicRoute
│   │   ├── HomePage
│   │   ├── Auth (Keycloak redirect)
│   │   └── NotFoundPage
│   │
│   └── ProtectedRoute (Auth required)
│       ├── Dashboard
│       │   ├── RoadmapGrid
│       │   ├── StatsCards
│       │   └── RecentActivity
│       │
│       ├── ViewRoadmapPage
│       │   ├── RoadmapViewer (ReactFlow)
│       │   ├── NodeDetailPanel
│       │   ├── ProgressTracker
│       │   └── ResourceList
│       │
│       ├── EditorRoadmapPage (Teacher only)
│       │   ├── RoadmapEditor (ReactFlow)
│       │   ├── Toolbar
│       │   ├── NodeEditor
│       │   ├── EdgeEditor
│       │   └── PropertiesPanel
│       │
│       ├── MultiChoiceQuizPage
│       │   ├── QuizPlayer
│       │   ├── QuestionCard
│       │   ├── Timer
│       │   └── ResultsPanel
│       │
│       ├── Learning (Teacher only)
│       │   ├── QuizBuilder
│       │   ├── QuizList
│       │   └── QuizSettings
│       │
│       └── Account
│           ├── ProfileForm
│           ├── PasswordChange
│           └── Preferences
```

#### Component Design Patterns

**1. Container/Presentational Pattern**

```tsx
// Container (smart component)
function DashboardContainer() {
  const { data, isLoading } = useGetMyRoadmapsQuery({ page: 0, size: 10 });
  const user = useAppSelector(selectUser);

  if (isLoading) return <LoadingSpinner />;

  return <DashboardView roadmaps={data} user={user} />;
}

// Presentational (dumb component)
function DashboardView({ roadmaps, user }: DashboardViewProps) {
  return (
    <div>
      <h1>Welcome, {user.name}</h1>
      <RoadmapGrid roadmaps={roadmaps} />
    </div>
  );
}
```

**2. Compound Components**

```tsx
// Parent component manages shared state
function RoadmapEditor({ roadmapId }: Props) {
  const [selectedNode, setSelectedNode] = useState<Node | null>(null);

  return (
    <EditorContext.Provider value={{ selectedNode, setSelectedNode }}>
      <Editor.Canvas />
      <Editor.Toolbar />
      <Editor.PropertiesPanel />
    </EditorContext.Provider>
  );
}
```

**3. Render Props**

```tsx
function QueryWrapper<T>({
  query,
  children,
}: {
  query: UseQueryResult<T>;
  children: (data: T) => ReactNode;
}) {
  const { data, isLoading, error } = query;

  if (isLoading) return <LoadingSpinner />;
  if (error) return <ErrorMessage error={error} />;
  if (!data) return <NoData />;

  return <>{children(data)}</>;
}

// Usage
<QueryWrapper query={useGetRoadmapQuery(id)}>
  {(roadmap) => <RoadmapViewer roadmap={roadmap} />}
</QueryWrapper>;
```

### 2. State Management Layer

#### Redux Store Structure

```typescript
// store/index.ts
const store = configureStore({
  reducer: {
    // Feature slices
    auth: authReducer,
    roadmap: roadmapReducer,
    quiz: quizReducer,
    ui: uiReducer,

    // RTK Query API slices
    [roadmapApi.reducerPath]: roadmapApi.reducer,
    [aiApi.reducerPath]: aiApi.reducer,
    [quizApi.reducerPath]: quizApi.reducer,
    [profileApi.reducerPath]: profileApi.reducer,
    [fileApi.reducerPath]: fileApi.reducer,
  },
  middleware: (getDefaultMiddleware) =>
    getDefaultMiddleware()
      .concat(roadmapApi.middleware)
      .concat(aiApi.middleware)
      .concat(quizApi.middleware)
      .concat(profileApi.middleware)
      .concat(fileApi.middleware),
});

export type RootState = ReturnType<typeof store.getState>;
export type AppDispatch = typeof store.dispatch;
```

#### Auth Slice Example

```typescript
// store/slices/authSlice.ts
interface AuthState {
  user: User | null;
  accessToken: string | null;
  refreshToken: string | null;
  isAuthenticated: boolean;
  isLoading: boolean;
}

const authSlice = createSlice({
  name: 'auth',
  initialState: {
    user: null,
    accessToken: localStorage.getItem('accessToken'),
    refreshToken: localStorage.getItem('refreshToken'),
    isAuthenticated: false,
    isLoading: true,
  } as AuthState,
  reducers: {
    setCredentials: (
      state,
      action: PayloadAction<{
        user: User;
        accessToken: string;
        refreshToken: string;
      }>,
    ) => {
      state.user = action.payload.user;
      state.accessToken = action.payload.accessToken;
      state.refreshToken = action.payload.refreshToken;
      state.isAuthenticated = true;
      state.isLoading = false;

      localStorage.setItem('accessToken', action.payload.accessToken);
      localStorage.setItem('refreshToken', action.payload.refreshToken);
    },
    logout: (state) => {
      state.user = null;
      state.accessToken = null;
      state.refreshToken = null;
      state.isAuthenticated = false;

      localStorage.removeItem('accessToken');
      localStorage.removeItem('refreshToken');
    },
    setLoading: (state, action: PayloadAction<boolean>) => {
      state.isLoading = action.payload;
    },
  },
});
```

### 3. Routing Layer

#### Route Configuration

```tsx
// routes/AppRoute.tsx
export const AppRoute = () => {
  return (
    <Routes>
      {/* Public Routes */}
      <Route path="/" element={<HomePage />} />
      <Route path="/auth" element={<AuthCallback />} />

      {/* Protected Routes (Authenticated users) */}
      <Route element={<ProtectedRoute />}>
        <Route path="/dashboard" element={<Dashboard />} />
        <Route path="/roadmap/:id" element={<ViewRoadmapPage />} />
        <Route path="/quiz/:id" element={<MultiChoiceQuizPage />} />
        <Route path="/account" element={<Account />} />
      </Route>

      {/* Teacher-only Routes */}
      <Route element={<ProtectedRoute requiredRole="TEACHER" />}>
        <Route path="/editor/:id" element={<EditorRoadmapPage />} />
        <Route path="/learning" element={<Learning />} />
      </Route>

      {/* 404 */}
      <Route path="*" element={<NotFoundPage />} />
    </Routes>
  );
};
```

#### Route Guard Implementation

```tsx
// routes/ProtectedRoute.tsx
interface ProtectedRouteProps {
  requiredRole?: 'STUDENT' | 'TEACHER';
}

export const ProtectedRoute = ({ requiredRole }: ProtectedRouteProps) => {
  const { isAuthenticated, user, isLoading } = useAppSelector(selectAuth);
  const location = useLocation();

  // Still checking auth status
  if (isLoading) {
    return <LoadingScreen />;
  }

  // Not authenticated
  if (!isAuthenticated) {
    return <Navigate to="/auth" state={{ from: location }} replace />;
  }

  // Check role if required
  if (requiredRole && user?.role !== requiredRole) {
    return <Navigate to="/dashboard" replace />;
  }

  return <Outlet />;
};
```

### 4. Service Layer

#### API Service Architecture

```
API Layer:
┌─────────────────────────────────────────────┐
│         RTK Query (Caching & State)         │
│  - Auto caching                             │
│  - Tag-based invalidation                   │
│  - Optimistic updates                       │
│  - Polling & refetching                     │
└─────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────┐
│         Axios Client (HTTP Transport)       │
│  - Request/response interceptors            │
│  - Token refresh logic                      │
│  - Error handling                           │
│  - Request cancellation                     │
└─────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────┐
│         API Gateway (Port 8280)             │
└─────────────────────────────────────────────┘
```

#### Axios Configuration

```typescript
// services/http/axiosClient.ts
const axiosClient = axios.create({
  baseURL: import.meta.env.VITE_API_GATEWAY_URL || 'http://localhost:8280',
  timeout: 30000,
  headers: {
    'Content-Type': 'application/json',
  },
});

// Request interceptor
axiosClient.interceptors.request.use(
  (config) => {
    const token = store.getState().auth.accessToken;
    if (token) {
      config.headers['Authorization'] = `Bearer ${token}`;
    }
    return config;
  },
  (error) => Promise.reject(error),
);

// Response interceptor
axiosClient.interceptors.response.use(
  (response) => response,
  async (error) => {
    const originalRequest = error.config;

    // Token expired, try refresh
    if (error.response?.status === 401 && !originalRequest._retry) {
      originalRequest._retry = true;

      try {
        const refreshToken = store.getState().auth.refreshToken;
        const response = await refreshAccessToken(refreshToken);

        store.dispatch(
          setCredentials({
            user: store.getState().auth.user!,
            accessToken: response.accessToken,
            refreshToken: response.refreshToken,
          }),
        );

        originalRequest.headers['Authorization'] =
          `Bearer ${response.accessToken}`;
        return axiosClient(originalRequest);
      } catch (refreshError) {
        store.dispatch(logout());
        window.location.href = '/auth';
        return Promise.reject(refreshError);
      }
    }

    return Promise.reject(error);
  },
);
```

## Key Technical Decisions

### 1. React Flow for Roadmap Visualization

**Why React Flow?**

- Built specifically for node-based UIs
- Excellent performance with large graphs
- Built-in zoom, pan, minimap
- Easy customization of nodes and edges
- TypeScript support

**Implementation:**

```tsx
function RoadmapViewer({ nodes, edges }: Props) {
  const [reactFlowNodes, setNodes] = useNodesState(transformNodes(nodes));
  const [reactFlowEdges, setEdges] = useEdgesState(transformEdges(edges));

  return (
    <ReactFlow
      nodes={reactFlowNodes}
      edges={reactFlowEdges}
      nodeTypes={customNodeTypes}
      edgeTypes={customEdgeTypes}
      onNodeClick={handleNodeClick}
      fitView
    >
      <Background />
      <Controls />
      <MiniMap />
    </ReactFlow>
  );
}
```

### 2. Redux Toolkit + RTK Query

**Why RTK Query?**

- Eliminates boilerplate for API calls
- Built-in caching reduces network requests
- Automatic refetching on focus/reconnect
- Tag-based cache invalidation
- Optimistic updates support
- Eliminates need for Redux Thunk

**Cache Strategy:**

```typescript
// Aggressive caching for static data
getCategoryById: builder.query({
  query: (id) => `/category/${id}`,
  keepUnusedDataFor: 300 // 5 minutes
}),

// Short-lived cache for dynamic data
getMyRoadmaps: builder.query({
  query: (params) => '/roadmap/my-roadmap',
  keepUnusedDataFor: 60 // 1 minute
}),

// No caching for real-time data
getProgress: builder.query({
  query: (roadmapId) => `/roadmap-tracking-progress/${roadmapId}`,
  keepUnusedDataFor: 0 // No cache
})
```

### 3. Tiptap Rich Text Editor

**Why Tiptap?**

- Headless and fully customizable
- Built on ProseMirror (powerful editing framework)
- Extensible with custom nodes/marks
- Good TypeScript support
- Rich feature set (tables, code blocks, etc.)

### 4. Tailwind CSS

**Why Tailwind?**

- Utility-first approach speeds development
- Purges unused CSS in production
- Consistent design system
- Easy responsive design
- JIT compiler for fast builds

## Performance Optimizations

### 1. Code Splitting

```tsx
// Lazy load heavy components
const EditorRoadmapPage = lazy(() => import('./pages/EditorRoadmapPage'));
const MultiChoiceQuizPage = lazy(() => import('./pages/MultiChoiceQuizPage'));

// Use Suspense for loading state
<Suspense fallback={<LoadingScreen />}>
  <EditorRoadmapPage />
</Suspense>;
```

### 2. Memoization

```tsx
// Memoize expensive computations
const transformedNodes = useMemo(() => {
  return nodes.map((node) => ({
    ...node,
    data: transformNodeData(node.data),
  }));
}, [nodes]);

// Memoize components
const NodeCard = memo(({ node }: Props) => {
  return <div>{node.label}</div>;
});
```

### 3. Virtual Scrolling

```tsx
// For long lists
import { Virtuoso } from 'react-virtuoso';

function RoadmapList({ roadmaps }: Props) {
  return (
    <Virtuoso
      data={roadmaps}
      itemContent={(index, roadmap) => (
        <RoadmapCard key={roadmap.id} roadmap={roadmap} />
      )}
    />
  );
}
```

### 4. Image Optimization

```tsx
// Lazy load images
<img
  src={roadmap.imageUrl}
  loading="lazy"
  alt={roadmap.title}
/>

// Use responsive images
<img
  srcSet={`
    ${roadmap.imageUrl}?w=320 320w,
    ${roadmap.imageUrl}?w=640 640w,
    ${roadmap.imageUrl}?w=1280 1280w
  `}
  sizes="(max-width: 640px) 320px, (max-width: 1024px) 640px, 1280px"
  alt={roadmap.title}
/>
```

## Security Considerations

### 1. XSS Prevention

- All user input is sanitized before rendering
- DOMPurify for rich text content
- CSP headers configured in production

### 2. CSRF Protection

- SameSite cookies
- CSRF tokens for state-changing operations

### 3. Secure Token Storage

- Access tokens: sessionStorage (short-lived)
- Refresh tokens: httpOnly cookies (recommended) or localStorage with encryption

### 4. Role-Based Access Control

```tsx
// Protect routes
<ProtectedRoute requiredRole="TEACHER">
  <EditorPage />
</ProtectedRoute>;

// Hide UI elements
{
  user?.role === 'TEACHER' && <CreateQuizButton />;
}

// Validate on backend (primary security)
```

## Build & Deployment

### Build Configuration

```typescript
// vite.config.ts
export default defineConfig({
  plugins: [react()],
  build: {
    outDir: 'dist',
    sourcemap: false,
    minify: 'terser',
    rollupOptions: {
      output: {
        manualChunks: {
          vendor: ['react', 'react-dom', 'react-router-dom'],
          ui: ['@radix-ui/react-dialog', '@radix-ui/react-dropdown-menu'],
          editor: ['reactflow', '@tiptap/react'],
          redux: ['@reduxjs/toolkit', 'react-redux'],
        },
      },
    },
  },
});
```

### Deployment Pipeline

```yaml
# .github/workflows/deploy.yml
name: Deploy Frontend

on:
  push:
    branches: [main]

jobs:
  build-deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '20'

      - name: Install dependencies
        run: npm ci

      - name: Run tests
        run: npm test

      - name: Build
        run: npm run build
        env:
          VITE_API_GATEWAY_URL: ${{ secrets.API_GATEWAY_URL }}
          VITE_KEYCLOAK_URL: ${{ secrets.KEYCLOAK_URL }}

      - name: Deploy to Vercel
        uses: amondnet/vercel-action@v20
        with:
          vercel-token: ${{ secrets.VERCEL_TOKEN }}
          vercel-org-id: ${{ secrets.VERCEL_ORG_ID }}
          vercel-project-id: ${{ secrets.VERCEL_PROJECT_ID }}
```

## Future Enhancements

1. **WebSocket Integration**: Real-time collaboration on roadmaps
2. **Offline Support**: Service Workers and IndexedDB for offline functionality
3. **Progressive Web App**: Installable app with push notifications
4. **Internationalization**: Multi-language support with i18next
5. **Analytics**: User behavior tracking with Google Analytics or Mixpanel
6. **Error Boundary**: Better error handling with Sentry integration
7. **Performance Monitoring**: Real User Monitoring (RUM) with Vercel Analytics
8. **A/B Testing**: Feature flags with LaunchDarkly or similar
9. **Accessibility**: WCAG 2.1 AA compliance improvements
10. **Mobile App**: React Native version for iOS/Android
