# TMS REST API Documentation

**Base URL:** `http://localhost:8080`  
**Content type:** `application/json` for JSON request/response bodies.  
**Authentication:** Authenticate using `POST /auth/login`. Send the access token using the mechanism expected by the frontend/gateway configuration for protected endpoints. The gateway validates JWTs and adds verified `X-User-*` identity headers for downstream services. Do not call internal service endpoints from the browser.


## Conventions

- Dates use `YYYY-MM-DD`.
- Roles: `ADMIN`, `PROJECT_MANAGER`, `TEAM_MEMBER`.
- Task statuses: `TODO`, `IN_PROGRESS`, `IN_REVIEW`, `DONE`.
- Task priorities: `LOW`, `MEDIUM`, `HIGH`, `CRITICAL`.
- Project statuses: `PLANNING`, `ACTIVE`, `ON_HOLD`, `COMPLETED`, `ARCHIVED`.
- Typical errors include `400 Bad Request` (invalid input), `401 Unauthorized`, `403 Forbidden`, `404 Not Found`, `409 Conflict`, `429 Too Many Requests` (login rate limit), and `503 Service Unavailable`. Error responses use a common shape with fields such as `timestamp`, `status`, `error`, `message`, `path`, and `fieldErrors`.

## 1. Authentication

| Method | Endpoint | Purpose | Access |
|---|---|---|---|
| POST | `/auth/login` | Authenticate and obtain tokens | Public |
| POST | `/auth/refresh` | Refresh/rotate tokens | Refresh-token holder |
| POST | `/auth/logout` | Log out / invalidate refresh session | Authenticated session |
| GET | `/auth/validate` | Validate the current token | Authenticated |

### `POST /auth/login`

Request:
```json
{
  "email": "admin@tms.com",
  "password": "Admin@123"
}
```

### `POST /auth/refresh`

Uses the refresh request DTO. Send the refresh token in the request body as required by the client implementation.

### `POST /auth/logout` and `GET /auth/validate`

Use the authentication state/token expected by the configured client. Consult `AuthHandler` and the frontend auth service for the precise token transport and response fields.

## 2. Users

| Method | Endpoint | Purpose | Access |
|---|---|---|---|
| GET | `/users/me` | Get current user's profile | Authenticated |
| PUT | `/users/me` | Update current user's profile | Authenticated |
| PUT | `/users/me/password` | Change password | Authenticated |
| GET | `/users/directory?q={query}&role={role}` | Search/filter user directory | Admin / project manager |
| POST | `/admin/users` | Create user | Admin |
| GET | `/admin/users` | List users (supports paging/filtering in implementation) | Admin |
| GET | `/admin/users/{id}` | Get a user | Admin |
| PUT | `/admin/users/{id}` | Update user, role, department, active state | Admin |
| DELETE | `/admin/users/{id}` | Deactivate user | Admin |

### `POST /admin/users`

```json
{
  "name": "Jordan Lee",
  "email": "jordan.lee@example.com",
  "password": "ChangeMe123!",
  "role": "TEAM_MEMBER",
  "phone": "9000000000",
  "department": "Engineering"
}
```

Required fields: `name`, valid `email`, `password` (8–64 characters), and `role`. `phone` and `department` are optional.

### `PUT /admin/users/{id}`

```json
{
  "name": "Jordan Lee",
  "role": "PROJECT_MANAGER",
  "phone": "9000000000",
  "department": "Engineering",
  "active": true
}
```

### `PUT /users/me` and `PUT /users/me/password`

Use the profile/password DTOs from the backend. Do not include an admin role change in the self-profile update; role assignment is an admin operation.

## 3. Projects

| Method | Endpoint | Purpose | Access |
|---|---|---|---|
| POST | `/projects` | Create a project | Admin / project manager |
| GET | `/projects` | List/search projects | Authenticated; results scoped by permissions |
| GET | `/projects/{id}` | Get project details | Authorized project viewer |
| PUT | `/projects/{id}` | Update project | Admin / owning project manager |
| PATCH | `/projects/{id}/archive` | Archive project | Admin / owning project manager |
| GET | `/projects/{id}/summary` | Project progress summary | Authorized project viewer |
| GET | `/projects/{id}/members` | List project members | Authorized project viewer |
| POST | `/projects/{id}/members` | Add project members | Admin / owning project manager |
| DELETE | `/projects/{id}/members/{userId}` | Remove a project member | Admin / owning project manager |

### `POST /projects`

```json
{
  "name": "Website Revamp",
  "description": "Redesign the corporate website",
  "startDate": "2026-10-29",
  "endDate": "2026-12-28",
  "managerId": 2,
  "memberIds": [3, 4]
}
```

`managerId` is optional (defaults to the caller for project managers); `memberIds` is an optional set of initial member IDs.

### `PUT /projects/{id}`

```json
{
  "name": "Website Revamp",
  "description": "Redesign the corporate website",
  "startDate": "2026-10-29",
  "endDate": "2026-12-28",
  "status": "ACTIVE",
  "managerId": 2
}
```

`managerId` is an admin-only reassignment field. Valid statuses: `PLANNING`, `ACTIVE`, `ON_HOLD`, `COMPLETED`, `ARCHIVED`.

### `POST /projects/{id}/members`

```json
{
  "userIds": [3, 4]
}
```

### `GET /projects`

Supported query parameters include `q`, `status`, `includeArchived`, `page`, `size`, and `sort`. Use the backend defaults if omitted.

## 4. Tasks

| Method | Endpoint | Purpose | Access |
|---|---|---|---|
| POST | `/tasks` | Create a task | Admin / project manager / eligible project member per service rules |
| GET | `/tasks/my` | List tasks assigned to current user | Authenticated |
| GET | `/tasks/project/{projectId}` | List/filter tasks in a project | Authorized project member |
| GET | `/tasks/project/{projectId}/board` | Kanban board data | Authorized project member |
| GET | `/tasks/project/{projectId}/summary` | Task completion summary | Authorized project viewer |
| GET | `/tasks/{id}` | Get task details | Authorized user |
| PUT | `/tasks/{id}` | Update task fields | Admin / owning project manager / task creator as allowed |
| DELETE | `/tasks/{id}` | Delete task (not when DONE) | Admin / owning project manager / task creator as allowed |
| PATCH | `/tasks/{id}/status` | Update task status | Admin / owning project manager / assigned member as allowed |
| PUT | `/tasks/{id}/assignee` | Assign or unassign a task | Authorized project manager/admin |
| GET | `/tasks/{id}/history` | View task audit history | Authorized project viewer |
| GET | `/tasks/{id}/comments` | List comments | Authorized project member |
| POST | `/tasks/{id}/comments` | Add a comment | Authorized project member |

### `POST /tasks`

```json
{
  "projectId": 1,
  "title": "Design login page",
  "description": "Prepare wireframes and final UI",
  "priority": "HIGH",
  "dueDate": "2026-11-05",
  "assigneeId": 3
}
```

`projectId` and `title` are required. `priority` defaults to `MEDIUM` when omitted. `assigneeId` is optional but, if supplied, the user must be a member of the project.

### `PUT /tasks/{id}`

```json
{
  "title": "Design login page",
  "description": "Prepare wireframes and final UI",
  "priority": "HIGH",
  "dueDate": "2026-11-05"
}
```

### `PATCH /tasks/{id}/status`

```json
{
  "status": "IN_PROGRESS"
}
```

### `PUT /tasks/{id}/assignee`

```json
{
  "assigneeId": 3
}
```

Set `assigneeId` to `null` to unassign the task.

### `POST /tasks/{id}/comments`

```json
{
  "content": "Please follow the updated design guidelines."
}
```

### `GET /tasks/project/{projectId}`

Supported query parameters: `status`, `priority`, `assigneeId`, `q`, `page`, `size`, and `sort`.

## 5. Monitoring endpoints

These endpoints are available on individual service ports and are not routed through the gateway:

| Method | Endpoint | Purpose |
|---|---|---|
| GET | `/monitoring/ping` | Basic liveness check |
| GET | `/monitoring/status` | Service, database, uptime, memory, and circuit-breaker status |
| GET | `/monitoring/circuit-breakers` | Circuit-breaker information |

Spring Boot Actuator endpoints are also configured per service (availability depends on each service configuration), including health and metrics.

## 6. Internal service endpoints

Routes under `/internal/**` are used for service-to-service communication and are **not routed by the API gateway**. They are intentionally omitted from the public API catalogue. Do not expose these endpoints directly to untrusted clients.

## 7. Demo login accounts

For local demonstration only:

| Email | Password | Role |
|---|---|---|
| `admin@tms.com` | `Admin@123` | `ADMIN` |
| `manager@tms.com` | `Manager@123` | `PROJECT_MANAGER` |
| `member1@tms.com` | `Member@123` | `TEAM_MEMBER` |
| `member2@tms.com` | `Member@123` | `TEAM_MEMBER` |

## 8. Suggested test sequence

1. `POST /auth/login`
2. `GET /users/me`
3. `POST /projects`
4. `GET /projects`
5. `POST /projects/{id}/members`
6. `POST /tasks`
7. `GET /tasks/project/{projectId}/board`
8. `PATCH /tasks/{id}/status`
9. `POST /tasks/{id}/comments`
10. `GET /tasks/{id}/history`

Use the gateway base URL for all public endpoints: `http://localhost:8080`.
