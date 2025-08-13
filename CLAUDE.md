# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a **Pickleball Club Management System** — a comprehensive multi-club platform for managing memberships, sessions/RSVP, payments, tournaments, and administrative controls. The system supports both web and mobile applications with a cloud-based architecture.
See @readme.md for more details.

## Technology Stack

### Backend (Planned)
- **Framework**: ASP.NET Core 8 with Minimal APIs/Controllers, C# 12
- **Database**: PostgreSQL with EF Core 8 + Npgsql provider
- **Authentication**: JWT-based with Google OAuth, Apple Sign-In, SMS OTP
- **Testing**: xUnit + FluentAssertions + NSubstitute/Moq
- **Validation**: FluentValidation for DTOs
- **Error Handling**: ProblemDetails (RFC 7807)

### Frontend (Planned)
- **Framework**: Angular 18+ with Standalone APIs
- **UI Library**: Angular Material (M3) or Tailwind CSS
- **Testing**: Jasmine/Karma (unit) + Playwright (e2e)
- **State Management**: Signals/NgRx with query-first approach
- **Mobile**: Responsive design with PWA capabilities

### Infrastructure
- **Cloud**: Azure
- **File Storage**: Azure Blob Storage
- **Payments**: Stripe integration for tournament fees
- **Notifications**: SMS via OTP, email, push notifications

## System Architecture

### Core Entities & Relationships
- **Users** with multiple authentication methods (Google, Apple, SMS)
- **Clubs** containing venues and courts
- **Members** with club-specific memberships and wallet balances
- **Sessions** with RSVP system and check-in functionality
- **Tournaments** with divisions, brackets, matches, and Stripe payments
- **Financial tracking** via wallet transactions and club-level finance management

### Key Features
1. **Multi-club support** with role-based permissions (Super Admin, Ledger Admin, Club Member)
2. **Session management** with voting/RSVP system and automated charging
3. **Tournament system** with bracket generation and live scoring
4. **Financial management** with individual wallets and club-level expense tracking
5. **Mobile-first design** with offline capability

## Development Commands

> **Note**: This project is in the planning/architecture phase. Some commands may be placeholders until implementation exists.

### Backend (.NET)
```bash
# Build and test
dotnet build
dotnet test
dotnet run --project ./src/Api/Api.csproj

# Database migrations
dotnet ef migrations add <Name> --project ./src/Infrastructure --startup-project ./src/Api
dotnet ef database update --project ./src/Infrastructure --startup-project ./src/Api
```

### Frontend (Angular)
```bash
# Development and testing
cd ./web
npm run start
npm run build
npm run test
npm run e2e
npm run lint
```

## Key Architecture Files
- `README.md` — Comprehensive requirements and technical specifications
- `architecture/openapi-specs.yaml` — REST API specification
- `architecture/entity-relationship.mmd` — Database schema as Mermaid ERD
- `tasks/backend-task-details.md` — ASP.NET Core implementation tasks
- `tasks/angular-frontend-tasks-specs.md` — Angular implementation specifications
- `tasks/frontend-tasks-overview.md` — Frontend task summaries with API mappings

## Authentication & Security
- **OAuth 2.0** with Google and Apple providers
- **SMS OTP** for phone-based verification
- **JWT tokens** with refresh token rotation
- **Role-based access control** with club-specific permissions
- **Rate limiting** and security headers for API protection

## Database Schema
PostgreSQL database with key tables:
- `users`, `user_identities`, `user_sessions` — Authentication
- `clubs`, `venues`, `courts` — Location management
- `members`, `memberships` — Club membership
- `sessions`, `rsvps` — Session and RSVP management
- `wallets`, `wallet_transactions` — Financial tracking
- `tournaments`, `divisions`, `brackets`, `matches` — Tournament system

## Development Guidelines
### Code Quality
- Follow strict typing and modern C#/TypeScript patterns
- Implement comprehensive unit tests (80%+ coverage target)
- Use FluentValidation for request validation
- Implement proper error handling with ProblemDetails
- Follow REST API conventions from OpenAPI spec

### Frontend Patterns
- Use Angular standalone components and signals
- Implement responsive, mobile-first design
- Handle loading, error, and empty states consistently
- Use optimistic updates for better UX
- Implement offline capability with caching

### Security Practices
- Never store tokens in localStorage without proper protection
- Implement proper CORS and rate limiting
- Use parameterized queries to prevent SQL injection
- Validate all user inputs on both client and server
- Implement audit logging for sensitive operations

---

# NEW: Containerization & Local Orchestration (Docker)

> These Docker assets enable consistent local dev and form the base for our Azure deployment.

## Directory layout
```
/docker
  /api/Dockerfile
  /web/Dockerfile
  compose.yaml
```

## API Dockerfile (`/docker/api/Dockerfile`)
```dockerfile
# --- build stage ---
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src
# copy csproj first for restore cache
COPY ./src/Api/*.csproj ./src/Api/
COPY ./src/Application/*.csproj ./src/Application/
COPY ./src/Domain/*.csproj ./src/Domain/
COPY ./src/Infrastructure/*.csproj ./src/Infrastructure/
RUN dotnet restore ./src/Api/Api.csproj

# copy remainder
COPY . .
RUN dotnet publish ./src/Api/Api.csproj -c Release -o /app/publish /p:UseAppHost=false

# --- runtime stage ---
FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS runtime
WORKDIR /app
ENV ASPNETCORE_URLS=http://+:8080
# connection string should be passed at runtime via env variable
ENV ConnectionStrings__Default="Host=postgres;Port=5432;Database=appdb;Username=appuser;Password=appsecret"
COPY --from=build /app/publish .
EXPOSE 8080
ENTRYPOINT ["dotnet", "Api.dll"]
```

## Web Dockerfile (`/docker/web/Dockerfile`)
```dockerfile
# --- build stage ---
FROM node:20-alpine AS build
WORKDIR /web
COPY ./web/package*.json ./
RUN npm ci
COPY ./web .
RUN npm run build

# --- nginx serve stage ---
FROM nginx:1.27-alpine
COPY --from=build /web/dist/ /usr/share/nginx/html
# Optional: custom nginx.conf for SPA routing (fallback to index.html)
# COPY ./docker/web/nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
```

## Compose for local dev (`/docker/compose.yaml`)
```yaml
version: "3.9"
services:
  postgres:
    image: postgres:16
    environment:
      POSTGRES_DB: appdb
      POSTGRES_USER: appuser
      POSTGRES_PASSWORD: appsecret
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U appuser -d appdb"]
      interval: 10s
      timeout: 5s
      retries: 5

  api:
    build:
      context: ..
      dockerfile: ./docker/api/Dockerfile
    environment:
      ASPNETCORE_ENVIRONMENT: "Development"
      ConnectionStrings__Default: "Host=postgres;Port=5432;Database=appdb;Username=appuser;Password=appsecret"
    ports:
      - "8080:8080"
    depends_on:
      postgres:
        condition: service_healthy

  web:
    build:
      context: ..
      dockerfile: ./docker/web/Dockerfile
    ports:
      - "4200:80"
    depends_on:
      - api

volumes:
  pgdata: {}
```

**Local run**
```bash
cd ./docker
docker compose up --build
# API: http://localhost:8080
# Web: http://localhost:4200
```

---

# NEW: Azure Deployment with Bicep + Azure Developer CLI (azd)

We use **Azure Developer CLI (azd)** to provision Azure resources using **Bicep** and deploy the containers built from the Dockerfiles above.

## Prereqs
- Azure CLI and Azure Developer CLI installed
- Logged in: `azd auth login` (or `az login`)
- `docker` logged in to push to ACR if building locally

## Repository layout for infra
```
/infra
  main.bicep                 # orchestration
  /modules
    acr.bicep
    containerapps.bicep
    postgres.bicep
    keyvault.bicep
    insights.bicep
  /params
    dev.bicepparam
    prod.bicepparam
azure.yaml                   # azd template descriptor (root)
```

## `azure.yaml` (root)
```yaml
name: pickleball-club
metadata:
  template: custom
infra:
  provider: bicep
  path: infra

services:
  api:
    project: ./src/Api
    host: containerapp
    language: dotnet
    docker:
      path: ./docker/api/Dockerfile
      context: .
    # Map infra outputs into runtime env for the container
    env:
      - name: ConnectionStrings__Default
        value: ${AZD_OUTPUT_POSTGRES_CONNECTION_STRING}
      - name: ASPNETCORE_ENVIRONMENT
        value: "Production"

  web:
    project: ./web
    host: containerapp
    language: js
    docker:
      path: ./docker/web/Dockerfile
      context: .
```

## Bicep — `infra/main.bicep` (orchestrator)
```bicep
targetScope = 'resourceGroup'

@description('App Name prefix')
param appName string
@description('Location')
param location string = resourceGroup().location
@description('Environment name (e.g., dev, prod)')
param env string = 'dev'

// ACR
module acr './modules/acr.bicep' = {
  name: 'acr'
  params: {
    name: '${appName}${env}acr'
    location: location
    sku: 'Basic'
  }
}

// Log Analytics + App Insights
module insights './modules/insights.bicep' = {
  name: 'insights'
  params: {
    appName: appName
    location: location
  }
}

// PostgreSQL Flexible Server
module pg './modules/postgres.bicep' = {
  name: 'postgres'
  params: {
    name: '${appName}-${env}-pg'
    location: location
    adminUser: 'pgadmin'
    adminPassword: '' // set via azd env and key vault if desired
    databaseName: 'appdb'
  }
}

// Key Vault
module kv './modules/keyvault.bicep' = {
  name: 'keyvault'
  params: {
    name: '${appName}-${env}-kv'
    location: location
  }
}

// Container Apps environment + two apps (api, web)
module ca './modules/containerapps.bicep' = {
  name: 'containerapps'
  params: {
    appName: appName
    env: env
    location: location
    containerRegistryLoginServer: acr.outputs.loginServer
    logsCustomerId: insights.outputs.workspaceId
    logsSharedKey: insights.outputs.primarySharedKey
    apiImage: '${acr.outputs.loginServer}/${appName}-api:${env}'
    webImage: '${acr.outputs.loginServer}/${appName}-web:${env}'
    postgresConnectionString: pg.outputs.connectionString
  }
}

// Expose outputs for azd -> service env mapping
output POSTGRES_CONNECTION_STRING string = pg.outputs.connectionString
output ACR_LOGIN_SERVER string = acr.outputs.loginServer
```

### Module stubs (examples)

`/infra/modules/acr.bicep`
```bicep
param name string
param location string
param sku string = 'Basic'

resource acr 'Microsoft.ContainerRegistry/registries@2023-01-01-preview' = {
  name: name
  location: location
  sku: { name: sku }
  properties: {
    adminUserEnabled: true
  }
}

output loginServer string = acr.properties.loginServer
```

`/infra/modules/postgres.bicep`
```bicep
param name string
param location string
param adminUser string
@secure()
param adminPassword string
param databaseName string

resource pg 'Microsoft.DBforPostgreSQL/flexibleServers@2023-12-01-preview' = {
  name: name
  location: location
  sku: { name: 'Standard_B1ms', tier: 'Burstable' }
  properties: {
    administratorLogin: adminUser
    administratorLoginPassword: adminPassword
    version: '16'
    storage: { storageSizeGB: 128 }
    backup: { backupRetentionDays: 7 }
    network: { publicNetworkAccess: 'Enabled' }
  }
}

resource db 'Microsoft.DBforPostgreSQL/flexibleServers/databases@2023-12-01-preview' = {
  name: '${pg.name}/${databaseName}'
  properties: {}
}

output connectionString string = 'Host=${pg.properties.fullyQualifiedDomainName};Port=5432;Database=${databaseName};Username=${adminUser};Password=${adminPassword};Ssl Mode=Require;'
```

`/infra/modules/insights.bicep`
```bicep
param appName string
param location string

resource la 'Microsoft.OperationalInsights/workspaces@2022-10-01' = {
  name: '${appName}-law'
  location: location
  sku: { name: 'PerGB2018' }
}

resource ai 'microsoft.insights/components@2020-02-02' = {
  name: '${appName}-ai'
  location: location
  kind: 'web'
  properties: {
    Application_Type: 'web'
    WorkspaceResourceId: la.id
  }
}

output workspaceId string = la.properties.customerId
output primarySharedKey string = la.listKeys().primarySharedKey
output appInsightsConnectionString string = ai.properties.ConnectionString
```

`/infra/modules/keyvault.bicep`
```bicep
param name string
param location string

resource kv 'Microsoft.KeyVault/vaults@2023-07-01' = {
  name: name
  location: location
  properties: {
    sku: { family: 'A', name: 'standard' }
    tenantId: tenant()
    accessPolicies: []
    enabledForDeployment: true
    enableRbacAuthorization: true
  }
}

output name string = kv.name
```

`/infra/modules/containerapps.bicep`
```bicep
param appName string
param env string
param location string
param containerRegistryLoginServer string
param logsCustomerId string
param logsSharedKey string
param apiImage string
param webImage string
param postgresConnectionString string

resource caEnv 'Microsoft.App/managedEnvironments@2024-02-02-preview' = {
  name: '${appName}-${env}-cae'
  location: location
  properties: {
    appLogsConfiguration: {
      destination: 'log-analytics'
      logAnalyticsConfiguration: {
        customerId: logsCustomerId
        sharedKey: logsSharedKey
      }
    }
  }
}

resource api 'Microsoft.App/containerApps@2024-02-02-preview' = {
  name: '${appName}-${env}-api'
  location: location
  properties: {
    managedEnvironmentId: caEnv.id
    configuration: {
      ingress: {
        external: true
        targetPort: 8080
      }
      registries: [
        {
          server: containerRegistryLoginServer
        }
      ]
      secrets: []
      activeRevisionsMode: 'Single'
    }
    template: {
      containers: [
        {
          name: 'api'
          image: apiImage
          env: [
            {
              name: 'ConnectionStrings__Default'
              value: postgresConnectionString
            }
            {
              name: 'ASPNETCORE_ENVIRONMENT'
              value: 'Production'
            }
          ]
        }
      ]
    }
  }
}

resource web 'Microsoft.App/containerApps@2024-02-02-preview' = {
  name: '${appName}-${env}-web'
  location: location
  properties: {
    managedEnvironmentId: caEnv.id
    configuration: {
      ingress: {
        external: true
        targetPort: 80
      }
      registries: [
        {
          server: containerRegistryLoginServer
        }
      ]
      activeRevisionsMode: 'Single'
    }
    template: {
      containers: [
        {
          name: 'web'
          image: webImage
        }
      ]
    }
  }
}

output apiUrl string = api.properties.configuration.ingress.fqdn
output webUrl string = web.properties.configuration.ingress.fqdn
```

### Environment parameters
Create `infra/params/dev.bicepparam`:
```bicep
using './main.bicep'

param appName = 'pickleball'
param env = 'dev'
```

## Build, provision & deploy with azd

### 1) Initialize the environment
```bash
azd init --template .
azd env new dev
azd auth login
```

### 2) Set secrets (Postgres password, etc.)
```bash
# For Bicep secure param adminPassword and for app runtime (if needed)
azd env set POSTGRES_ADMIN_PASSWORD <StrongPassword>
# Provide value to the Bicep param via --parameters when provisioning:
# azd provision --parameters adminPassword=$POSTGRES_ADMIN_PASSWORD
```

### 3) Provision Azure resources
```bash
azd provision --environment dev --parameters adminPassword=$POSTGRES_ADMIN_PASSWORD
```

### 4) Build & push containers, deploy to Container Apps
By default azd will build and push using the Dockerfiles specified in `azure.yaml`.
```bash
azd deploy --environment dev
```

### 5) Get output endpoints
```bash
azd env get-values    # shows outputs exported from Bicep
# Or check the deployed service URLs in the final azd output
```

### 6) Optional: CI/CD pipeline
```bash
azd pipeline config   # set up GitHub Actions or Azure DevOps
```

### 7) Tear down (clean up the resource group)
```bash
azd down
```

---

## Observability
- Local: console logging; use structured JSON logging if desired.
- Cloud: Application Insights for API (extend Bicep to inject the connection string); Log Analytics captures container logs.

## Guardrails (Claude)
Add `.claude/settings.json` to protect secrets and prod config:
```json
{
  "permissions": {
    "deny": [
      "Read(./.env)",
      "Read(./.env.*)",
      "Read(./secrets/**)",
      "Read(./infra/params/prod*.bicepparam)",
      "Write(./infra/params/prod*.bicepparam)"
    ]
  }
}
```

## Shortcuts for Claude
- “Generate EF migration for <change>” → update `Infrastructure` and suggest `dotnet ef` commands.
- “Create Angular component <name>`” → add to `/web/src/app/...` with routing & tests.
- “Build Docker images & run locally” → run `docker compose up --build` in `/docker`.
- “Provision Azure & deploy” → run `azd provision` then `azd deploy` with the active env.