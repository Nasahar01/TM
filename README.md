# Team Project & Task Management System 

A full-stack team, project, and task management application. The frontend is Angular; the backend uses Java 17, Spring Boot WebFlux, Spring Cloud Gateway, R2DBC/MySQL, JWT authentication, Eureka service discovery, and Resilience4j.

## 1. Main modules

- **Authentication:** login, token refresh, logout, and token validation.
- **User management:** user directory, profile updates, password changes, and admin user/role management.
- **Projects:** create and browse projects, update/archive projects, manage project members, and view progress summaries.
- **Tasks:** create, assign, update, and track tasks; Kanban board; comments; task history; project-level filtering and summaries.
- **Security:** JWT validation at the API gateway, role-based access, and BCrypt password hashing.

## 2. Backend architecture

| Component | Default port | Purpose |
|---|---:|---|
| API Gateway | 8080 | Public entry point; JWT validation and routing |
| Auth service | 8081 | Login, refresh, logout, token validation |
| User service | 8082 | User profiles, directory, roles, admin management |
| Project service | 8083 | Projects and project membership |
| Task service | 8084 | Tasks, board, comments, and history |
| Eureka server | 8761 | Service discovery; run separately |

The gateway routes requests to the services through Eureka. Internal `/internal/**` endpoints are for service-to-service calls and are not exposed through the gateway.

## 3. Prerequisites

- Java 17
- Maven (or use the Maven wrapper if included)
- Node.js and npm compatible with the Angular version in `tms-frontend-src`
- MySQL 8.0
- A Eureka server available at `http://localhost:8761/eureka`

## 4. Database setup

From the backend root directory, start MySQL with Docker Compose if desired:

```bash
docker compose up -d
```

Or start a local MySQL 8 instance and execute `database_setup_and_sample_data.sql` in stages as described below.

1. Run the database-creation statements at the beginning of `database_setup_and_sample_data.sql` (the `CREATE DATABASE` statements). These create `authtms`, `usertms`, `projecttms`, and `tasktms`.
2. Configure `DB_HOST`, `DB_PORT`, `DB_USERNAME`, and `DB_PASSWORD` if your MySQL settings differ from the defaults.
3. Start the services once. Each service creates its own tables from its `src/main/resources/schema.sql`; the user service also seeds the demo users.
4. If you want demo projects and tasks, run the optional sample-data statements at the end of the SQL file **once only**, after the tables exist and the seeded user IDs match the assumptions documented in the SQL file.


## 5. Start the backend

Open the `pom.xml` from each backend service as a Maven project, ensure Java 17 is selected, and enable Lombok annotation processing in your IDE.

Start these applications:
1. Your Eureka server on port 8761.
2. `UserServiceApplication`
3. `AuthServiceApplication`
4. `ProjectServiceApplication`
5. `TaskServiceApplication`
6. `ApiGatewayApplication` (start the gateway after the services are registering)

Wait for the services to register in Eureka, then use the API through `http://localhost:8080`. Do not use service ports directly for normal frontend requests.

### Environment configuration

The backend supports these environment variables:

- `DB_HOST` — database host
- `DB_PORT` — database port
- `DB_USERNAME` — MySQL username
- `DB_PASSWORD` — MySQL password
- `EUREKA_URL` — Eureka endpoint (default is `http://localhost:8761/eureka`)
- `JWT_SECRET` — JWT signing secret; keep the same value in the gateway and auth service

Use a strong, private JWT secret outside local development. Never commit production credentials or secrets to GitHub.

## 6. Demo accounts

The user service seeds these development accounts on first startup:

| Email | Password | Role |
|---|---|---|
| `admin@tms.com` | `Admin@123` | `ADMIN` |
| `manager@tms.com` | `Manager@123` | `PROJECT_MANAGER` |
| `member1@tms.com` | `Member@123` | `TEAM_MEMBER` |
| `member2@tms.com` | `Member@123` | `TEAM_MEMBER` |

These are sample credentials for local demonstration only. Change or remove them before any production deployment.

## 7. Start the frontend

1. Extract `tms-frontend-src.zip`.
2. Open a terminal in the extracted frontend project directory (the directory containing `package.json`).
3. Install dependencies and start the Angular development server:

```bash
npm install
npm start
```

If the project does not define a `start` script, use:

```bash
npx ng serve
```

Open the local URL printed by Angular (commonly `http://localhost:4200`). The backend gateway allows the development frontend origin `http://localhost:4200`. Verify the frontend API base URL points to `http://localhost:8080`.

## 8. API documentation

See **`API_Documentation.md`** for the endpoint catalogue, methods, access rules, query parameters, and representative request bodies. All normal API requests should go through `http://localhost:8080`.

## 9. Roles and access rules

| Capability | Admin | Project Manager | Team Member |
|---|---|---|---|
| Manage users | Yes | No | No |
| Create projects | Yes | Yes | No |
| Edit/archive project and manage members | Any project | Own projects | No |
| View project | All projects | Own/member projects | Projects they belong to |
| Create/assign tasks | All projects | Own projects | Projects they belong to |
| Edit/delete task | All tasks, subject to workflow rules | Own projects | Tasks they created |
| Update task status | All tasks | Own projects | Tasks assigned to them |
| Comment | All projects | Managed/member projects | Member projects |

Additional workflow rules: tasks can only be assigned to project members; completed (`DONE`) tasks cannot be deleted; archived projects are read-only; task changes are recorded in task history.

## 10. Troubleshooting

- **Database connection failure:** confirm MySQL is running, the four databases exist, and database environment variables are correct.
- **Services are unavailable / 503:** ensure Eureka is running and wait for all services to register.
- **401 Unauthorized:** log in and send the access token as expected by the frontend/API client; call the gateway on port 8080.
- **Lombok-generated methods are unresolved:** install the Lombok IDE plugin and enable annotation processing.
- **Frontend requests fail:** confirm the backend gateway is running on port 8080 and the configured API base URL matches it.
- **Port conflict:** change the relevant service port in its `application.yml` and update any dependent configuration.

## 11. Tests

From the backend root, run:

```bash
mvn clean verify
```

This runs the backend module tests configured in the Maven project.
