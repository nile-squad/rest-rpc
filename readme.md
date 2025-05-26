# The "REST-RPC" Architecture - Implementation Guide - Current Draft

This document provides a complete overview of the "REST-RPC" API architecture implementation, a hybrid approach that centralizes service operations while maintaining REST-like discoverability.

Mostly useful for micro services that don't want to leave rest api land and backend for frontend systems (BFF) and service oriented designs or event driven systems. But see the when not to use section as well.

## 1. Introduction to "REST-RPC"

The "REST-RPC" architecture implemented here is a service-oriented approach that organizes APIs around **services** rather than resources. Each service exposes multiple **actions** through a standardized interface, combining the discoverability of REST with the explicit operation naming of RPC.

## 2. Core Architecture

### 2.1 Service-Based Organization

- APIs are organized around **services** (e.g., `users`, `products`, `orders`)
- Each service contains multiple **actions** (e.g., `create`, `update`, `delete`, `login`)
- Services can be manually defined or auto-generated from database schemas

### 2.2 URL Structure

```plaintext
{baseUrl}/{apiVersion}/services/{serviceName}
```

Example: `https://api.example.com/v1/services/users`

### 2.3 HTTP Methods

- **`GET`**: Service discovery and documentation
- **`POST`**: Action execution

## 3. Request/Response Format

### 3.1 POST Request Structure

All service actions use POST with this standardized body:

```js
{
  "resourceId": "optional_resource_identifier",
  "action": "action_name",
  "payload": {
    // Action-specific data
  }
}
```

### 3.2 Response Structure

All responses follow this format:

```js
{
  "status": true|false,
  "message": "Descriptive message",
  "data": { /* Response data */ }
}
```

## 4. Service Discovery Endpoints

### 4.1 List All Services

```plaintext
GET {baseUrl}/{apiVersion}/services
```

**Response:**

```js
{
  "status": true,
  "message": "List of all available services on ServerName.",
  "data": ["users", "products", "orders"]
}
```

### 4.2 Service Details

```plaintext
GET {baseUrl}/{apiVersion}/services/{serviceName}
```

**Response:**

```js
{
  "status": true,
  "message": "Service Details",
  "data": {
    "name": "users",
    "description": "User management service",
    "availableActions": ["create", "login", "updateProfile", "delete"]
  }
}
```

### 4.3 Action Details

```plaintext
GET {baseUrl}/{apiVersion}/services/{serviceName}/{actionName}
```

**Response:**

```js
{
  "status": true,
  "message": "Action Details",
  "data": {
    "name": "create",
    "description": "Creates a new user account",
    "validation": {
      /* JSON Schema for payload validation */
    }
  }
}
```

### 4.4 Complete Schema Documentation

```plaintext
GET {baseUrl}/{apiVersion}/services/schema
```

**Response:**

```js
{
  "status": true,
  "message": "ServerName Services actions zod Schemas",
  "data": [
    {
      "users": [
        {
          "name": "create",
          "description": "Creates a new user",
          "validation": { /* JSON Schema */ }
        }
      ]
    }
  ]
}
```

## 5. Configuration and Setup

### 5.1 Server Configuration

```typescript
import { useRestRPC } from './rest-rpc';

const config: ServerConfig = {
  serverName: "My API Server",
  baseUrl: "/api",
  apiVersion: "v1",
  authSecret: "your-jwt-secret",
  host: "0.0.0.0",
  port: "8000",
  services: [/* service definitions */],
  allowedOrigins: ["*"],
  enableStatic: true,
  enableStatus: true,
  rateLimiting: {
    windowMs: 15 * 60 * 1000, // 15 minutes
    limit: 100,
    limitingHeader: "x-forwarded-for"
  },
  db: {
    instance: dbInstance,
    tables: dbTables
  }
};

const app = useRestRPC(config);
```

### 5.2 Service Definition Structure

```typescript
type Service = {
  name: string;
  description: string;
  actions: Action[];
  autoService?: boolean; // Enable auto-generation
  subs?: Service[]; // Sub-services for auto-generation
};

type Action = {
  name: string;
  description: string;
  validation?: {
    zodSchema: ZodSchema;
  };
  isProtected?: boolean; // Requires JWT authentication
  handler: (payload: any, context: HonoContext) => Promise<any>;
};
```

## 6. Authentication

### 6.1 Protected Actions

Actions marked with `isProtected: true` require JWT authentication:

```http
POST /api/v1/services/users
Authorization: Bearer <jwt-token>
Content-Type: application/json

{
  "action": "updateProfile",
  "payload": {
    "name": "New Name"
  }
}
```

### 6.2 Authentication Flow

1. Client includes `Authorization: Bearer <token>` header
2. Server verifies JWT using configured `authSecret`
3. Invalid/missing tokens return 401 Unauthorized

## 7. Auto-Generated Services

### 7.1 Database Integration

Services can be auto-generated from database schemas:

```typescript
const service: Service = {
  name: "users",
  description: "User management",
  autoService: true,
  actions: [], // Manual actions
  subs: [
    {
      name: "profiles",
      description: "User profiles",
      actions: [] // Will be auto-generated
    }
  ]
};
```

### 7.2 Generated Actions

Auto-services typically generate standard CRUD operations:

- `create` - Create new records
- `read` - Retrieve records
- `update` - Update existing records
- `delete` - Delete records
- `list` - List records with pagination/filtering

## 8. Rate Limiting

### 8.1 Configuration

```typescript
rateLimiting: {
  windowMs: 15 * 60 * 1000, // Time window
  limit: 100, // Max requests per window
  limitingHeader: "x-forwarded-for", // Header to extract client ID
  standardHeaders: true, // Include rate limit headers in response
  store: redisStore // Optional: Redis store for distributed limiting
}
```

### 8.2 Rate Limit Headers

Responses include standard rate limiting headers:

- `RateLimit-Limit`: Maximum requests allowed
- `RateLimit-Remaining`: Requests remaining in window
- `RateLimit-Reset`: Time when window resets

## 9. Error Handling

### 9.1 Validation Errors

```json
{
  "status": false,
  "message": "Invalid request format",
  "data": {
    "field": "email",
    "message": "Invalid email format"
  }
}
```

### 9.2 Action Not Found

```json
{
  "status": false,
  "message": "Action 'invalidAction' not found in service 'users'",
  "data": {}
}
```

### 9.3 Authentication Errors

```json
{
  "status": false,
  "message": "Unauthorized",
  "data": {}
}
```

## 10. Additional Features

### 10.1 Static File Serving

When `enableStatic: true`:

```http
GET /assets/image.png
```

Serves files from `./assets/` directory.

### 10.2 Health Check

When `enableStatus: true`:

```http
GET /status
```

**Response:**

```json
{
  "status": true,
  "message": "ServerName Server running well 😎",
  "data": {}
}
```

### 10.3 CORS Support

Automatic CORS handling for all origins (configurable via `allowedOrigins`).

## 11. Example Implementation

### 11.1 Complete Service Example

```typescript
const userService: Service = {
  name: "users",
  description: "User management service",
  actions: [
    {
      name: "create",
      description: "Create a new user account",
      validation: {
        zodSchema: z.object({
          email: z.string().email(),
          password: z.string().min(8),
          name: z.string()
        })
      },
      handler: async (payload, context) => {
        // Implementation
        return {
          status: true,
          message: "User created successfully",
          data: { id: "123", email: payload.email }
        };
      }
    },
    {
      name: "login",
      description: "Authenticate user",
      validation: {
        zodSchema: z.object({
          email: z.string().email(),
          password: z.string()
        })
      },
      handler: async (payload, context) => {
        // Authentication logic
        return {
          status: true,
          message: "Login successful",
          data: { token: "jwt-token" }
        };
      }
    }
  ]
};
```

### 11.2 Client Usage Example

```javascript
// Create user
const response = await fetch('/api/v1/services/users', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    action: 'create',
    payload: {
      email: 'user@example.com',
      password: 'password123',
      name: 'John Doe'
    }
  })
});

// Login
const loginResponse = await fetch('/api/v1/services/users', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    action: 'login',
    payload: {
      email: 'user@example.com',
      password: 'password123'
    }
  })
});
```

## 12. Benefits of This Implementation

1. **Centralized Action Handling**: All service operations go through a single endpoint
2. **Strong Typing**: Zod schema validation with automatic JSON Schema generation
3. **Auto-Discovery**: Complete API documentation available at runtime
4. **Flexible Authentication**: Per-action authentication control
5. **Auto-Generation**: Database-driven service generation
6. **Rate Limiting**: Built-in protection against abuse
7. **Standardized Responses**: Consistent error handling and response format

## 13. When to Use This Architecture

**Ideal For:**

- Microservices with complex business logic
- APIs requiring strong validation and documentation
- Systems needing flexible authentication per operation
- Applications with database-driven CRUD operations
- Internal APIs where explicit action naming improves clarity

**Consider Alternatives When:**

- Building simple REST APIs with standard CRUD operations
- Public APIs where REST conventions are expected
- Systems requiring HTTP method-based caching strategies
- Applications needing hypermedia-driven discovery (HATEOAS)

This implementation provides a robust, scalable foundation for service-oriented APIs with excellent developer experience through comprehensive documentation and validation.

## Final Notes

This is still an experimental architecture, only tested with implementation of hono js, drizzle orm, bun js, and postgres + sqlite. There may be variations in implementation depending on the framework and database used. Also Typescript is strongly recommended.

## License & Contributing

This architecture is still experimental and subject to change. So contributions, criticism and feedback are welcome.

Created by [Hussein Kizz](https://github.com/Hussseinkizz) at Nile Squad Labz. Completely open source under MIT License and used in production.

