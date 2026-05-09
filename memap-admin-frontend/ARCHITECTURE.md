# Admin Frontend Architecture

## Role in the System

The Admin Frontend is a separate Next.js application from the user-facing `memap-frontend`. It is used exclusively by platform administrators and uses Keycloak SSO instead of the custom JWT cookie authentication used in the main app.

## Application Structure

```
memap-admin-frontend/
├── app/
│   ├── layout.tsx                   # Root layout (Keycloak + Antd providers)
│   ├── page.tsx                     # Landing / redirect
│   ├── forbidden/page.tsx           # 403 page
│   └── (protected)/                 # All routes require Keycloak session
│       ├── layout.tsx               # Protected layout guard
│       └── admin/
│           ├── layout.tsx           # Admin shell (AdminLayout)
│           ├── page.tsx             # Dashboard
│           ├── teachers/
│           │   ├── page.tsx                             # Teacher list
│           │   └── [teacherId]/roadmaps/
│           │       ├── page.tsx                         # Teacher's roadmaps
│           │       └── [roadmapId]/storages/page.tsx    # Roadmap file storage
│           ├── students/page.tsx                        # Student list
│           ├── invitations/page.tsx                     # Invitation management
│           └── roadmap-categories/page.tsx              # Category management
├── app/services/                    # API client modules
│   ├── studentService.ts
│   ├── teacherService.ts
│   ├── profileUserService.ts
│   ├── roadmapService.ts
│   └── storageService.ts
├── app/components/                  # Shared UI components
│   ├── AdminLayout.tsx
│   ├── DataTable.tsx
│   ├── UserFormDialog.tsx
│   ├── DeleteConfirmDialog.tsx
│   ├── ExcelImportDialog.tsx
│   └── PageContent.tsx
├── lib/
│   ├── keycloak.ts                  # Keycloak client instance
│   ├── keycloakAuth.ts              # Token refresh utilities
│   └── authRoutes.ts                # Protected route config
├── providers/
│   ├── ReactKeycloakProvider.tsx    # Keycloak context provider
│   └── AntdProvider.tsx            # Ant Design theme provider
└── constant/
    └── api-config.ts                # API Gateway base URL
```

## Authentication Flow

```
User visits admin.memap.id.vn
         │
         ▼
ReactKeycloakProvider initializes Keycloak.js
         │
         ├─ Not authenticated → redirect to Keycloak login page
         └─ Authenticated → user lands in protected (protected)/ layout
                                     │
                             All API calls attach:
                             Authorization: Bearer <keycloak.token>
                             (refreshed via keycloak.updateToken(30) before each call)
```

## API Call Pattern

All service modules follow the same pattern:

```typescript
await keycloak.updateToken(30);        // refresh if < 30s remaining
header.Authorization = `Bearer ${keycloak.token}`;

fetch(`${API_GATEWAY_BASE_URL}/profile/...`, { headers: header })
```

The `API_GATEWAY_BASE_URL` resolves to (in order):
1. `NEXT_PUBLIC_PROFILE_BASE_URL` (env var, trailing slash stripped)
2. `NEXT_PUBLIC_API_BASE_URL` (env var, trailing slash stripped)
3. `http://localhost:8090` (development default)

## Difference from memap-frontend

| Aspect              | memap-frontend                  | memap-admin-frontend             |
| ------------------- | ------------------------------- | -------------------------------- |
| Auth provider       | Profile Service JWT (cookies)   | Keycloak SSO (Bearer token)      |
| Framework           | React (Vite)                    | Next.js 14 (App Router)          |
| UI library          | Custom / Tailwind               | Ant Design + Tailwind            |
| Target users        | Students & Teachers             | Platform Administrators          |
| Real-time features  | SSE, Hocuspocus collab editing  | None (admin CRUD only)           |
| Production URL      | `roadmap.memap.id.vn`           | `admin.memap.id.vn`              |

## Backend Services Consumed

| Service           | API Gateway path | Operations                                  |
| ----------------- | ---------------- | ------------------------------------------- |
| Profile Service   | `/profile`       | User list, roles, invitations               |
| Roadmap Service   | `/roadmap`       | Read teacher roadmaps                       |
| Storage Service   | `/storage`       | Browse roadmap file storage                 |
