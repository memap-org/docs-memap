# MeMap Frontend

## Overview

The MeMap Frontend is a modern React-based single-page application (SPA) built with TypeScript, Vite, and Redux Toolkit. It provides an intuitive user interface for creating, editing, viewing, and managing learning roadmaps, as well as taking quizzes and tracking progress.

## Key Features

- **Dashboard**: Personalized dashboard with roadmap overview and progress tracking
- **Roadmap Viewer**: Interactive roadmap visualization with nodes and edges
- **Roadmap Editor**: Visual editor for creating and editing roadmaps (Teacher-only)
- **Quiz System**: Take quizzes and view results
- **Learning Page**: Browse and discover learning resources
- **Profile Management**: User profile and settings
- **Authentication**: Login/logout with Keycloak integration
- **Role-Based Access**: Different features for Students and Teachers
- **Responsive Design**: Mobile-friendly interface
- **Rich Text Editing**: Tiptap editor for content creation

## Technology Stack

- **Framework**: React 18.3
- **Language**: TypeScript 5.6
- **Build Tool**: Vite 6.0
- **State Management**: Redux Toolkit with RTK Query
- **Routing**: React Router v6
- **UI Framework**: Custom components with Tailwind CSS
- **Styling**: Tailwind CSS 3.4
- **Rich Text Editor**: Tiptap
- **HTTP Client**: Axios (via RTK Query)
- **Package Manager**: npm/pnpm

## Project Structure

```
memap-frontend/
├── public/
│   ├── data.json
│   └── images/
├── src/
│   ├── assets/           # Static assets (images, fonts)
│   ├── components/       # Reusable React components
│   │   ├── common/      # Common UI components
│   │   ├── roadmap/     # Roadmap-specific components
│   │   ├── quiz/        # Quiz components
│   │   └── ...
│   ├── features/        # Feature-based modules
│   │   ├── auth/       # Authentication
│   │   ├── roadmap/    # Roadmap management
│   │   ├── quiz/       # Quiz functionality
│   │   └── profile/    # User profile
│   ├── pages/          # Page components
│   │   ├── Dashboard.tsx
│   │   ├── HomePage.tsx
│   │   ├── ViewRoadmapPage.tsx
│   │   ├── EditorRoadmapPage.tsx
│   │   ├── MultiChoiceQuizPage.tsx
│   │   ├── Learning.tsx
│   │   └── ...
│   ├── routes/         # Routing configuration
│   │   ├── AppRoute.tsx
│   │   └── ProtectedRoute.tsx
│   ├── store/          # Redux store configuration
│   ├── services/       # API services (RTK Query)
│   ├── hooks/          # Custom React hooks
│   ├── utils/          # Utility functions
│   ├── types/          # TypeScript type definitions
│   ├── constants/      # Constants and configuration
│   ├── App.tsx         # Root component
│   ├── main.tsx        # Entry point
│   └── index.css       # Global styles
├── package.json
├── tsconfig.json
├── vite.config.ts
├── tailwind.config.js
└── vercel.json
```

## Getting Started

### Prerequisites

- Node.js 18+ or higher
- npm or pnpm

### Installation

```bash
# Navigate to frontend directory
cd memap-frontend

# Install dependencies
npm install
# or
pnpm install
```

### Development

```bash
# Start development server
npm run dev
# or
pnpm dev

# Access at http://localhost:5173
```

### Build

```bash
# Build for production
npm run build
# or
pnpm build

# Preview production build
npm run preview
```

### Linting

```bash
# Run ESLint
npm run lint
```

## Configuration

### Environment Variables

Create a `.env` file in the root directory:

```env
VITE_API_BASE_URL=http://localhost:8280
VITE_KEYCLOAK_URL=http://localhost:8080
VITE_KEYCLOAK_REALM=memap
VITE_KEYCLOAK_CLIENT_ID=memap-frontend
```

### Vite Configuration

Key configurations in `vite.config.ts`:

```typescript
export default defineConfig({
  plugins: [react()],
  server: {
    port: 5173,
    proxy: {
      '/api': {
        target: 'http://localhost:8280',
        changeOrigin: true,
      },
    },
  },
});
```

## Routing

### Public Routes

- `/` - Home page
- `/about` - About Us page
- `/auth` - Login page
- `/learning` - Learning resources

### Protected Routes (Authenticated Users)

- `/dashboard` - User dashboard (Student, Teacher)
- `/billing` - Billing and subscriptions (Student, Teacher)
- `/account/:section` - Account settings (Student, Teacher)
- `/roadmap/:id` - View roadmap (Student, Teacher)
- `/roadmap/:id/quiz` - Take quiz (Student, Teacher)

### Teacher-Only Routes

- `/roadmap/edt/:id` - Edit roadmap (Teacher)

### Route Configuration

```tsx
// src/routes/AppRoute.tsx
function AppRoute() {
  return (
    <Routes>
      <Route path="/" element={<Layout />}>
        <Route index element={<HomePage />} />
        <Route path="/learning" element={<Learning />} />
        <Route path="/about" element={<AboutUs />} />
        <Route path="/auth" element={<Login />} />

        <Route
          path="/dashboard"
          element={
            <ProtectedRoute
              requiredRoles={[USER_ROLES.STUDENT, USER_ROLES.TEACHER]}
            >
              <Dashboard />
            </ProtectedRoute>
          }
        />

        <Route
          path="/roadmap/edt/:id"
          element={
            <ProtectedRoute requiredRoles={[USER_ROLES.TEACHER]}>
              <EditorRoadmapPage />
            </ProtectedRoute>
          }
        />
      </Route>
    </Routes>
  );
}
```

## State Management

### Redux Store Structure

```typescript
{
  auth: {
    user: User | null,
    token: string | null,
    isAuthenticated: boolean
  },
  roadmap: {
    currentRoadmap: Roadmap | null,
    myRoadmaps: Roadmap[],
    loading: boolean,
    error: string | null
  },
  quiz: {
    currentQuiz: Quiz | null,
    attempts: QuizAttempt[],
    loading: boolean
  },
  profile: {
    userProfile: Profile | null,
    loading: boolean
  }
}
```

### RTK Query API Services

```typescript
// src/services/api/roadmapApi.ts
export const roadmapApi = createApi({
  reducerPath: 'roadmapApi',
  baseQuery: fetchBaseQuery({
    baseUrl: `${API_BASE_URL}/roadmap-service/road-map/api`,
    prepareHeaders: (headers, { getState }) => {
      const token = (getState() as RootState).auth.token;
      if (token) {
        headers.set('authorization', `Bearer ${token}`);
      }
      return headers;
    },
  }),
  endpoints: (builder) => ({
    getRoadmap: builder.query<RoadmapResponse, string>({
      query: (id) => `/roadmap/${id}`,
    }),
    createRoadmap: builder.mutation<
      CreateRoadmapResponse,
      CreateRoadmapRequest
    >({
      query: (body) => ({
        url: '/roadmap',
        method: 'POST',
        body,
      }),
    }),
    updateNode: builder.mutation({
      query: ({ roadmapId, nodeRequest }) => ({
        url: `/roadmap/node/${roadmapId}`,
        method: 'PATCH',
        body: nodeRequest,
      }),
    }),
  }),
});
```

## Key Pages

### 1. Home Page (`HomePage.tsx`)

- Landing page with product overview
- Feature highlights
- Call-to-action buttons
- Navigation to dashboard or login

### 2. Dashboard (`Dashboard.tsx`)

- Overview of user's roadmaps
- Progress tracking widgets
- Recent activity
- Quick actions (create roadmap, browse quizzes)
- Role-specific content (Teacher vs Student)

### 3. Roadmap Viewer (`ViewRoadmapPage.tsx`)

- Interactive node-edge visualization
- Node details panel
- Progress tracking
- AI-powered node summarization
- Quiz integration
- Share and collaborate features

**Key Features**:

- Zoom and pan controls
- Node click for details
- Progress indicators
- Responsive layout

### 4. Roadmap Editor (`EditorRoadmapPage.tsx`)

**Teacher-only**

- Visual drag-and-drop editor
- Add/edit/delete nodes
- Create connections (edges)
- Rich text editing for node content
- Image upload for roadmap thumbnail
- Save and publish

**Key Features**:

- Real-time preview
- Undo/redo support
- Auto-save
- Validation

### 5. Quiz Page (`MultiChoiceQuizPage.tsx`)

- Display quiz questions
- Multiple-choice options
- Timer (if configured)
- Submit answers
- View results and explanations

### 6. Learning Page (`Learning.tsx`)

- Browse available roadmaps
- Filter by category
- Search functionality
- Roadmap cards with preview

### 7. Profile/Account (`Account.tsx`)

- User profile information
- Settings and preferences
- Change password
- Notification preferences

## Key Components

### ProtectedRoute

```tsx
function ProtectedRoute({
  children,
  requiredRoles,
}: {
  children: React.ReactNode;
  requiredRoles: string[];
}) {
  const { isAuthenticated, user } = useAuth();

  if (!isAuthenticated) {
    return <Navigate to="/auth" />;
  }

  if (
    requiredRoles &&
    !requiredRoles.some((role) => user.roles.includes(role))
  ) {
    return <Navigate to="/forbidden" />;
  }

  return <>{children}</>;
}
```

### RoadmapViewer Component

- Canvas-based rendering
- Node positioning algorithm
- Edge rendering with arrows
- Interactive tooltips

### RichTextEditor (Tiptap)

```tsx
import { useEditor, EditorContent } from '@tiptap/react';
import StarterKit from '@tiptap/starter-kit';

function RichTextEditor({ content, onChange }) {
  const editor = useEditor({
    extensions: [StarterKit],
    content,
    onUpdate: ({ editor }) => {
      onChange(editor.getHTML());
    },
  });

  return <EditorContent editor={editor} />;
}
```

## Authentication Flow

1. User clicks "Login"
2. Redirect to Keycloak login page
3. User enters credentials
4. Keycloak redirects back with authorization code
5. Frontend exchanges code for tokens
6. Store tokens in Redux + localStorage
7. Set up axios interceptor for token refresh
8. Redirect to dashboard

### Token Management

```typescript
// Axios interceptor for token refresh
axios.interceptors.response.use(
  (response) => response,
  async (error) => {
    if (error.response?.status === 401) {
      // Token expired, try to refresh
      const refreshToken = localStorage.getItem('refreshToken');
      if (refreshToken) {
        const newToken = await refreshAccessToken(refreshToken);
        // Retry original request with new token
        error.config.headers['Authorization'] = `Bearer ${newToken}`;
        return axios(error.config);
      } else {
        // Refresh failed, logout
        dispatch(logout());
      }
    }
    return Promise.reject(error);
  },
);
```

## Styling

### Tailwind CSS

- Utility-first CSS framework
- Custom theme configuration
- Responsive design utilities

### Tailwind Configuration

```javascript
// tailwind.config.js
module.exports = {
  content: ['./index.html', './src/**/*.{js,ts,jsx,tsx}'],
  theme: {
    extend: {
      colors: {
        primary: '#3B82F6',
        secondary: '#10B981',
        danger: '#EF4444',
      },
      fontFamily: {
        sans: ['Inter', 'sans-serif'],
      },
    },
  },
  plugins: [],
};
```

## API Integration

### Base API Configuration

```typescript
// src/constants/api-config.ts
export const API_BASE_URL =
  import.meta.env.VITE_API_BASE_URL || 'http://localhost:8280';

export const API_ENDPOINTS = {
  IDENTITY: '/identity-service/identity',
  PROFILE: '/profile-service/profile/api',
  ROADMAP: '/roadmap-service/road-map/api',
  FILE: '/file-service/file',
  AI: '/roadmap-ai-service',
  LEARNING: '/learning-service/learning/api/v1',
};
```

## Deployment

### Vercel Configuration

```json
{
  "buildCommand": "npm run build",
  "outputDirectory": "dist",
  "framework": "vite",
  "rewrites": [
    {
      "source": "/(.*)",
      "destination": "/index.html"
    }
  ]
}
```

### Build Optimization

- Code splitting with dynamic imports
- Lazy loading for routes
- Asset optimization
- Tree shaking

## Performance Optimization

### Code Splitting

```tsx
import { lazy, Suspense } from 'react';

const Dashboard = lazy(() => import('./pages/Dashboard'));

function App() {
  return (
    <Suspense fallback={<Loading />}>
      <Dashboard />
    </Suspense>
  );
}
```

### Memoization

```tsx
import { useMemo, useCallback } from 'react';

const MemoizedComponent = React.memo(({ data }) => {
  const processedData = useMemo(() => expensiveComputation(data), [data]);

  const handleClick = useCallback(() => {
    // Handle click
  }, []);

  return <div onClick={handleClick}>{processedData}</div>;
});
```

## Testing

### Unit Tests (To be implemented)

- Jest + React Testing Library
- Component tests
- Hook tests
- Utility function tests

### E2E Tests (To be implemented)

- Cypress or Playwright
- User flow tests
- Integration tests

## Browser Support

- Chrome (latest)
- Firefox (latest)
- Safari (latest)
- Edge (latest)
- Mobile browsers (iOS Safari, Chrome)

## Future Enhancements

1. **Progressive Web App (PWA)**: Offline support, installability
2. **Real-time Collaboration**: WebSocket for live editing
3. **Internationalization**: Multi-language support
4. **Dark Mode**: Theme switching
5. **Accessibility**: WCAG compliance
6. **Performance**: Further optimization with React 19
7. **Testing**: Comprehensive test coverage
8. **Analytics**: User behavior tracking
9. **Notifications**: Real-time push notifications
10. **Mobile App**: React Native version
