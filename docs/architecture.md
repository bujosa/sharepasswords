# Architecture

This document describes the system architecture of SharePassword.

## Overview

SharePassword is a monorepo containing multiple packages that work together to provide a secure password sharing platform.

```
sharepasswords/
├── packages/
│   ├── backend/         # NestJS API server
│   ├── frontend/        # Svelte 5 SPA
│   ├── core/            # Shared domain logic
│   ├── types/           # Shared TypeScript types
│   └── config/          # ESLint & TSConfig
├── docker-compose.yml   # Container orchestration
└── pnpm-workspace.yaml  # Monorepo configuration
```

## Technology Stack

| Layer | Technology | Purpose |
|-------|------------|---------|
| **Frontend** | Svelte 5 + Vite | Reactive UI with client-side encryption |
| **Backend** | NestJS 11 | REST API with modular architecture |
| **Database** | MongoDB | Document storage with TTL indexes |
| **Encryption** | Web Crypto API | AES-256-GCM encryption |
| **Build** | pnpm workspaces | Monorepo package management |

## System Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                          FRONTEND                                │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐         │
│  │  CreateSecret│    │  ViewSecret │    │   Router    │         │
│  │  Component   │    │  Component  │    │             │         │
│  └──────┬──────┘    └──────┬──────┘    └─────────────┘         │
│         │                  │                                     │
│  ┌──────▼──────────────────▼──────┐                             │
│  │         Crypto Library          │                             │
│  │  (AES-256-GCM, Web Crypto API) │                             │
│  └──────────────┬─────────────────┘                             │
│                 │                                                │
│  ┌──────────────▼─────────────────┐                             │
│  │         API Client              │                             │
│  └──────────────┬─────────────────┘                             │
└─────────────────┼───────────────────────────────────────────────┘
                  │ HTTPS (encrypted blob only)
                  │
┌─────────────────▼───────────────────────────────────────────────┐
│                          BACKEND                                 │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │                    API Layer (REST)                          ││
│  │  ┌─────────────────┐    ┌─────────────────┐                 ││
│  │  │ SecretController │    │ HealthController │                 ││
│  │  │  POST /secrets   │    │  GET /health     │                 ││
│  │  │  GET /secrets/:id│    └─────────────────┘                 ││
│  │  └────────┬────────┘                                         ││
│  └───────────┼──────────────────────────────────────────────────┘│
│              │                                                   │
│  ┌───────────▼──────────────────────────────────────────────────┐│
│  │                   Service Layer                               ││
│  │  ┌─────────────────┐                                         ││
│  │  │  SecretService   │                                         ││
│  │  │  - create()     │                                         ││
│  │  │  - getAndConsume│                                         ││
│  │  │  - exists()     │                                         ││
│  │  └────────┬────────┘                                         ││
│  └───────────┼──────────────────────────────────────────────────┘│
│              │                                                   │
│  ┌───────────▼──────────────────────────────────────────────────┐│
│  │                   Data Layer                                  ││
│  │  ┌─────────────────┐    ┌─────────────────┐                 ││
│  │  │  SecretSchema    │    │   MongoDB        │                 ││
│  │  │  (Mongoose ODM)  │◄──►│   (TTL indexes)  │                 ││
│  │  └─────────────────┘    └─────────────────┘                 ││
│  └──────────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────────┘
```

## Backend Architecture

### Module Structure

The backend follows NestJS's modular architecture:

```
backend/src/
├── main.ts                    # Application bootstrap
├── modules/
│   ├── app.module.ts          # Root module
│   ├── health/                # Health check module
│   │   └── api/
│   │       └── health.controller.ts
│   └── secret/                # Core feature module
│       ├── secret.module.ts
│       ├── api/rest/
│       │   ├── secret.controller.ts
│       │   └── dto/
│       │       ├── create-secret.dto.ts
│       │       └── get-secret.dto.ts
│       ├── core/services/
│       │   └── secret.service.ts
│       └── infra/db/
│           └── secret.schema.ts
├── db/
│   └── database.module.ts     # MongoDB configuration
├── core/
│   ├── config/                # Configuration management
│   └── context/               # Request context handling
└── infra/
    ├── api/
    │   ├── interceptors/      # HTTP interceptors
    │   ├── middlewares/       # Express middleware
    │   └── filter/            # Exception filters
    └── libs/logging/          # Pino logger
```

### Request Flow

1. **Request arrives** at `SecretController`
2. **DTO Validation** via Zod schemas
3. **Business Logic** in `SecretService`
4. **Database Operations** via Mongoose ODM
5. **Response** formatted and returned

## Frontend Architecture

### Component Structure

```
frontend/src/
├── main.ts                    # Entry point
├── App.svelte                 # Root component + router
├── lib/
│   ├── crypto.ts              # Encryption utilities
│   ├── api.ts                 # Backend API client
│   ├── router.ts              # SPA routing
│   └── qrcode.ts              # QR code generation
└── components/
    ├── Header.svelte          # Navigation
    ├── CreateSecret.svelte    # Secret creation UI
    ├── ViewSecret.svelte      # Secret viewing UI
    ├── DocsPage.svelte        # Documentation
    ├── ApiPage.svelte         # API reference
    └── SecurityPage.svelte    # Security info
```

### Routing

| Route | Component | Description |
|-------|-----------|-------------|
| `/` | CreateSecret | Home page, create new secrets |
| `/s/:secretId` | ViewSecret | View and decrypt a secret |
| `/docs` | DocsPage | Documentation |
| `/api` | ApiPage | API reference |
| `/security` | SecurityPage | Security information |
| `/privacy` | PrivacyPage | Privacy policy |
| `/terms` | TermsPage | Terms of service |

## Database Schema

### Secrets Collection

```javascript
{
  secretId: String,           // Unique identifier (UUID)
  encryptedContent: String,   // AES-256-GCM encrypted blob (base64)
  expiresAt: Date,            // TTL expiration timestamp
  maxViews: Number | null,    // Max allowed views (null = unlimited)
  currentViews: Number,       // Current view count
  viewedAt: Date | null,      // Last viewed timestamp
  createdAt: Date,            // Creation timestamp
  updatedAt: Date             // Last update timestamp
}
```

### Indexes

| Index | Type | Purpose |
|-------|------|---------|
| `secretId` | Unique | Fast lookups by ID |
| `expiresAt` | TTL | Automatic document expiration |

## Zero-Knowledge Design

The architecture ensures the server **never** has access to plaintext:

### Key Storage

```
URL: https://sharepasswords.com/s/abc123#encryption-key-here
                                        │
                                        └── Fragment (# and after)
                                            NEVER sent to server
```

### Data Flow

1. **Client generates** encryption key using Web Crypto API
2. **Client encrypts** plaintext with AES-256-GCM
3. **Client sends** only encrypted blob to server
4. **Server stores** encrypted blob (cannot decrypt)
5. **Client shares** URL with key in fragment
6. **Recipient extracts** key from URL fragment
7. **Recipient requests** encrypted blob from server
8. **Recipient decrypts** locally with key

## Expiration Strategy

### Time-Based Expiration

MongoDB TTL indexes automatically delete expired documents:

```javascript
// Schema index
{ expiresAt: 1 }, { expireAfterSeconds: 0 }
```

Expiration options:
- `1h` - 1 hour
- `24h` - 24 hours
- `7d` - 7 days
- `30d` - 30 days

### View-Based Expiration

Application-level enforcement:

```javascript
if (secret.maxViews !== null && secret.currentViews >= secret.maxViews) {
  await secret.deleteOne();
  return null;
}
```

## Deployment Architecture

### Docker Compose

```yaml
services:
  database:
    image: mongo:7
    volumes:
      - mongodb_data:/data/db

  backend:
    build: ./packages/backend
    depends_on:
      - database
    environment:
      - DATABASE_URL=mongodb://database:27017/sharepasswords

  frontend:
    build: ./packages/frontend
    depends_on:
      - backend
```

### Production Considerations

- **Load Balancing**: Multiple backend instances behind a load balancer
- **Database Replication**: MongoDB replica set for high availability
- **CDN**: Static frontend assets served via CDN
- **TLS**: All traffic encrypted with HTTPS
- **Rate Limiting**: API rate limiting to prevent abuse

## Design Decisions

### Why Svelte 5?

- Small bundle size (important for privacy-focused users)
- Excellent reactivity model
- No virtual DOM overhead
- Simple state management

### Why NestJS?

- Structured, modular architecture
- Built-in dependency injection
- Excellent TypeScript support
- Production-ready features (logging, validation, etc.)

### Why MongoDB?

- Native TTL index support for automatic cleanup
- Flexible document model
- Horizontal scalability
- Easy to operate

### Why URL Fragments for Keys?

- Fragments (`#`) are never sent to the server per HTTP spec
- Provides true zero-knowledge architecture
- No server-side changes needed
- Works with all browsers
