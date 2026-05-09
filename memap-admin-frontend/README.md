# MeMap Admin Frontend

## Overview

The Admin Frontend is a Next.js application for platform administrators. It provides management interfaces for teachers, students, roadmap categories, invitations, and teacher roadmaps with file storage. Authentication is handled by Keycloak (not the custom JWT cookie flow used by the main frontend).

## Technology Stack

- **Next.js 14+** (App Router, TypeScript)
- **Keycloak** (`@react-keycloak/web`) — SSO authentication
- **Ant Design** — UI component library
- **Tailwind CSS** — utility-first styling

## Authentication

Uses Keycloak directly (not the profile-service JWT flow):

- `lib/keycloak.ts` — Keycloak client configuration
- `lib/keycloakAuth.ts` — token refresh helpers
- `providers/ReactKeycloakProvider.tsx` — app-level Keycloak context
- Access token is attached as `Authorization: Bearer <token>` on every API call

All pages under `app/(protected)/` require an authenticated Keycloak session. Unauthenticated users are redirected to the Keycloak login page.

## Pages

| Route                                                    | Description                              |
| -------------------------------------------------------- | ---------------------------------------- |
| `/admin`                                                 | Admin dashboard                          |
| `/admin/teachers`                                        | List and manage teachers                 |
| `/admin/students`                                        | List and manage students                 |
| `/admin/invitations`                                     | Manage user invitations                  |
| `/admin/roadmap-categories`                              | Manage roadmap categories                |
| `/admin/teachers/[teacherId]/roadmaps`                   | View roadmaps owned by a teacher         |
| `/admin/teachers/[teacherId]/roadmaps/[roadmapId]/storages` | Browse roadmap file storage           |

## API Integration

All requests go through the API Gateway (`http://localhost:8090` by default):

| Service          | Base path          | env var                        |
| ---------------- | ------------------ | ------------------------------ |
| Profile Service  | `/profile`         | `NEXT_PUBLIC_API_BASE_URL`     |
| Roadmap Service  | `/roadmap`         | `NEXT_PUBLIC_API_BASE_URL`     |
| Storage Service  | `/storage`         | `NEXT_PUBLIC_API_BASE_URL`     |

`NEXT_PUBLIC_PROFILE_BASE_URL` takes precedence over `NEXT_PUBLIC_API_BASE_URL` if set.

## Environment Variables

| Variable                    | Default                   | Description                    |
| --------------------------- | ------------------------- | ------------------------------ |
| `NEXT_PUBLIC_API_BASE_URL`  | `http://localhost:8090`   | API Gateway base URL           |
| `NEXT_PUBLIC_PROFILE_BASE_URL` | (optional)             | Overrides API base URL         |
| `NEXT_PUBLIC_KEYCLOAK_URL`  | (required in production)  | Keycloak server URL            |
| `NEXT_PUBLIC_KEYCLOAK_REALM`| (required in production)  | Keycloak realm name            |
| `NEXT_PUBLIC_KEYCLOAK_CLIENT_ID` | (required in production) | Keycloak client ID        |

## Running Locally

```bash
cd memap-admin-frontend
yarn install
yarn dev
```

Production build:

```bash
yarn build && yarn start
```

Or via Docker:

```bash
docker-compose up memap-admin-frontend
```

Admin UI is accessible at `https://admin.memap.id.vn` in production.
