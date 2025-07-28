# The "REST-RPC" Architecture - Implementation Guide - Current Draft

This document provides a complete overview of the "REST-RPC" API architecture implementation, a hybrid approach that centralizes service operations while maintaining REST-like discoverability.

Mostly useful for micro services that don't want to leave rest api land and backend for frontend systems (BFF) and service oriented designs or event driven systems. But see the when not to use section as well and the Question and Answer Section too!

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
  host: "0.0.0.0",
  port: "8000",
  services: [/* service definitions */],
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
  isProtected?: boolean; // Requires authentication
  handler: (payload: any, context: HonoContext) => Promise<ActionResponse>;
};

type ActionResponse = {
  status: boolean;
  message: string;
  data: any;
};
```

**Handler Consistency Requirements:**

All action handlers must:
- Accept exactly two parameters: `(payload, context)`
- Return a Promise that resolves to an `ActionResponse` object
- Follow the standardized response format with `status`, `message`, and `data` fields
- Handle errors gracefully and return error responses in the same format

This ensures consistency across all actions at scale and predictability across teams.

## 6. Authentication

### 6.1 Protected Actions

Actions marked with `isProtected: true` require authentication:

```http
POST /api/v1/services/users
Authorization: Bearer <some-token>
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
2. Server verifies token before invoking action
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
      actions: [] // Will be auto-generated but supplemental actions can also be provided
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

## 8. Error Handling and Validation

### 8.1 Validation Errors

When payload validation fails:

```json
{
  "status": false,
  "message": "Invalid request format",
  "data": {
    "field": "email",
    "message": "Invalid email format",
    "code": "VALIDATION_ERROR"
  }
}
```

### 8.2 Action Not Found

When requesting a non-existent action:

```json
{
  "status": false,
  "message": "Action 'invalidAction' not found in service 'users'",
  "data": {
    "availableActions": ["create", "login", "updateProfile", "delete"],
    "code": "ACTION_NOT_FOUND"
  }
}
```

### 8.3 Authentication Errors

When authentication fails:

```json
{
  "status": false,
  "message": "Unauthorized",
  "data": {
    "code": "UNAUTHORIZED",
    "reason": "Invalid or missing token"
  }
}
```

### 8.4 Server Errors

When internal errors occur:

```json
{
  "status": false,
  "message": "Internal server error",
  "data": {
    "code": "INTERNAL_ERROR",
    "requestId": "req_123456789"
  }
}
```

### 8.5 Business Logic Errors

When business rules are violated:

```json
{
  "status": false,
  "message": "Insufficient balance for transaction",
  "data": {
    "code": "BUSINESS_RULE_VIOLATION",
    "currentBalance": 50.00,
    "requiredAmount": 100.00
  }
}
```

**Error Handling Patterns:**

- All errors follow the same response structure
- Include error codes for programmatic handling
- Provide contextual information in the `data` field
- Use descriptive messages for debugging
- Maintain consistent error categorization across services

## 9. Example Implementation

### 9.1 Complete Service Example

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
        try {
          // Business logic here
          const user = await createUser(payload);
          
          return {
            status: true,
            message: "User created successfully",
            data: { id: user.id, email: user.email }
          };
        } catch (error) {
          return {
            status: false,
            message: "Failed to create user",
            data: { 
              code: "USER_CREATION_FAILED",
              reason: error.message 
            }
          };
        }
      }
    }
  ]
};
```

### 9.2 Client Usage Example

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

const result = await response.json();
if (result.status) {
  console.log('User created:', result.data);
} else {
  console.error('Error:', result.message, result.data);
}
```

## 10. Benefits of This Implementation

1. **Centralized Action Handling**: All service operations go through a single endpoint
2. **Strong Typing**: Zod schema validation with automatic JSON Schema generation
3. **Auto-Discovery**: Complete API documentation available at runtime
4. **Flexible Authentication**: Per-action authentication control
5. **Auto-Generation**: Database-driven service generation (optional)
6. **Standardized Responses**: Consistent error handling and response format
7. **Standardized Request Format**: Consistent request structure for all services
8. **Handler Consistency**: Predictable action handler interface across all services

## 11. When to Use This Architecture

**Ideal For:**

- Microservices with complex business logic
- APIs requiring strong validation and documentation
- Systems needing flexible authentication per operation
- Applications with needs beyond database-driven CRUD operations
- Internal APIs where explicit action naming improves clarity

**Consider Alternatives When:**

- Building simple REST APIs with standard CRUD operations
- Public APIs where REST conventions are expected
- Systems requiring HTTP method-based caching strategies
- Applications needing hypermedia-driven discovery (HATEOAS)

This implementation provides a robust, scalable foundation for service-oriented APIs with excellent developer experience through comprehensive documentation and validation.

But if you're not convinced yet, for these concerns... consider:

## 12. Frequently Asked Questions

### 12.1 Architecture Decisions

**Q: Why not use GraphQL instead?**

A: While I appreciate GraphQL's power, I found it introduces too much learning curve for my frontend teams who are already productive with REST patterns. GraphQL requires learning query syntax, new tooling, different error handling, and complex caching strategies. My REST-RPC approach lets teams keep using familiar `fetch()` calls while still getting explicit operations and strong typing. Sometimes the path of least resistance is the right path.

**Q: Why not use tRPC for type safety?**

A: I actually love tRPC's type safety, but it forces you into a monorepo or complex type-sharing setup. Since I work with separate frontend/backend repositories (which is pretty common in enterprise), tRPC's main benefits disappear while adding infrastructure complexity. I wanted the type safety without the coupling, which is why I'm planning client magic using our Zod schemas - think tRPC-like inference without the monorepo requirement.

**Q: Why not stick to pure REST principles?**

A: I tried, but enterprise business logic just doesn't fit cleanly into HTTP verbs. When I have operations like `calculateShipping`, `processPayment`, or `generateReport`, forcing these into PUT/POST feels artificial. I'd rather be explicit about what the operation does than pretend it's a resource update. Business clarity trumps HTTP purity for internal systems.

**Q: Doesn't this violate HTTP semantics by using POST for everything?**

A: Absolutely, and I'm okay with that trade-off. I prioritize business clarity and consistent patterns over semantic correctness. For internal enterprise APIs where I control both ends, the benefits of explicit actions outweigh the loss of HTTP-level caching. I can always add caching at the application layer where it makes more sense anyway.

### 12.2 Standards and Conventions

**Q: How does this compare to JSON-RPC or other RPC standards?**

A: I borrowed from RPC concepts but kept REST-like discoverability because I wanted the best of both worlds:

| Feature       | JSON-RPC        | My REST-RPC                         | Traditional REST         |
| ------------- | --------------- | ----------------------------------- | ------------------------ |
| Discovery     | External docs   | Built-in `/services` endpoints      | HATEOAS (rarely works)   |
| HTTP Methods  | POST only       | GET for discovery, POST for actions | Full HTTP verb semantics |
| URL Structure | Single endpoint | Service-based endpoints             | Resource-based endpoints |
| Validation    | Manual          | Auto-generated from Zod schemas     | Manual or OpenAPI        |

**Q: What about OpenAPI/Swagger compatibility?**

A: I'm planning to add Swagger support in future versions, but honestly, I don't think it's as critical here. My `/services/schema` endpoint already provides machine-readable API docs, and since everything follows the same action pattern, there's less need for detailed endpoint documentation. When I do add Swagger, it'll focus on service/action details rather than trying to map everything to traditional REST patterns.

**Q: How do you handle caching without proper HTTP verbs?**

A: For enterprise internal APIs, I rely on application-level caching strategies - Redis, database query optimization, service-level caching. HTTP-level caching is often overrated for complex business logic anyway. Most enterprise operations involve multiple data sources and complex calculations that don't cache well at the HTTP layer.

**Q: Is this approach RESTful?**

A: Nope, not in the strict sense, and I'm not pretending it is. I violate the uniform interface constraint by not using HTTP verbs semantically. I call it "REST-RPC" because it borrows REST's discoverability while using RPC's explicit operations. It's a hybrid that works for my use cases, not a pure implementation of either.

### 12.3 Implementation Concerns

**Q: How do you handle idempotency without PUT/DELETE?**

A: I handle idempotency at the application level using the `resourceId` + `action` combination for deduplication, idempotency keys in payloads when needed, and database constraints. It's actually cleaner than trying to force business operations into HTTP verb semantics that don't quite fit.

**Q: What about performance and scalability?**

A: For internal enterprise APIs, I've found that business logic is usually the bottleneck, not HTTP routing. Single endpoints per service actually simplify load balancing, and I can scale services independently. The auto-generation features optimize for development speed over performance for basic crud, which is the right trade-off for most enterprise scenarios.

**Q: How do you version this API?**

A: I use multiple strategies:

- URL versioning for major changes: `/api/v1/services/users` → `/api/v2/services/users`
- Action evolution: Add new actions, deprecate old ones gradually
- Payload evolution: Add optional fields, maintain backward compatibility
- Service splitting: Break large services into focused ones

The action-based approach actually makes versioning easier since I can add new actions without breaking existing ones.

**Q: What about testing and mocking?**

A: The consistent request/response format actually makes testing simpler. One POST endpoint per service reduces mock complexity, standardized responses enable generic error handling, and action-based testing makes business logic cleaner to test. I can mock entire services with a single endpoint handler.

### 12.4 Future Plans and Framework Development

**Q: You mentioned client magic - what's that about?**

A: I'm working on client libraries that use the Zod schemas from `/services/schema` to provide tRPC-like type inference without monorepo coupling. Imagine getting full TypeScript autocomplete and validation on your frontend just by pointing to the schema endpoint - and some codegen magic, It's like having your cake and eating it too.

**Q: What about action effects and scheduling?**

A: I'm planning to add action effects (like triggering other actions after completion) and action scheduling (cron-like scheduling for actions). The action-based architecture makes these features natural extensions. Think of it as building a workflow engine on top of the service layer.

**Q: Any plans for real-time features?**

A: I'm considering WebSocket support for action subscriptions - imagine subscribing to action results or getting real-time updates when certain actions complete. The service/action model maps well to event-driven architectures.

**Q: Is this becoming a full framework?**

A: Absolutely! I'm working on turning this into a complete framework that teams can adopt without having to rebuild everything from scratch. The goal is to package all the patterns, tooling, and best practices I've developed into something that maintains the same quality and developer experience regardless of who implements it.

The framework will include:

- **CLI tools** for scaffolding new services and actions
- **Code generators** for common patterns (CRUD, auth, validation)
- **Client libraries** for multiple languages (TypeScript, Python, Go, Rust, etc)
- **Development tools** like service explorers and testing utilities
- **Deployment templates** for common platforms (Docker, Kubernetes, serverless)
- **Monitoring and observability** built-in from day one

**Q: How will you ensure quality doesn't degrade as it becomes a framework?**

A: I'm taking a few approaches to maintain quality:

1. **Battle-tested patterns**: Everything in the framework comes from real production usage, not theoretical ideals
2. **Comprehensive testing**: The framework will have extensive test suites and real-world validation
3. **Gradual rollout**: I'm starting with internal teams before open-sourcing, so I can catch issues early
4. **Strong defaults**: The framework will be opinionated about the right way to do things, reducing the chance for quality degradation
5. **Continuous dogfooding**: I'll keep using it for my own projects, so I'll feel the pain if quality drops

**Q: When will this framework be available?**

A: I'm targeting for internal use for now, with an open-source release following once I've validated it across multiple projects. I want to make sure it's genuinely useful and not just another "framework of the week."

The plan is to start with the core service/action patterns, then gradually add the client magic, scheduling, and advanced features. I'd rather ship something solid and simple than something complex and buggy. And no pressure here anyone can implement this architecture on their own in any language.

### 12.5 When to Use This Architecture

**Q: When should I NOT use this approach?**

A: Don't use this if you're building public APIs where REST conventions are expected, simple CRUD apps where REST mapping is natural, or performance-critical apps requiring HTTP caching. Also skip it if your team is already happy with GraphQL or if you're working across organizational boundaries where standards matter more than productivity.

**Q: What's the migration path from existing REST APIs?**

A: I recommend gradual migration:

1. Start new services with REST-RPC
2. Create adapter services wrapping existing REST endpoints
3. Migrate high-change services first
4. Keep stable CRUD services as-is
5. Use an API gateway for unified interface

**Q: How do you train teams on this approach?**

A: I tell them "it's just POST with a standard body format" and focus on thinking in business actions rather than HTTP verbs. The learning curve is minimal since it builds on familiar HTTP concepts. Most developers get it within a day because they're already thinking in terms of functions and operations anyway.

The key is showing them the `/services` endpoints for discovery and emphasizing the consistent error handling. Once they see how much boilerplate disappears, they're usually sold.

**Q: What if I want to contribute to the framework development?**

A: I'm always open to feedback and contributions! Right now I'm in the design and validation phase, so input on patterns, use cases, and pain points is incredibly valuable. Reason why I open-sourced it, I'll need help with a few things, client libraries for different languages, integrations with various databases and platforms, and real-world testing across different types of projects and so on.

The best way to contribute right now is to try the patterns in your own projects and share what works (and what doesn't). That real-world feedback is what will make this framework genuinely useful rather than just another mental exercise.

## Final Notes

This is still an experimental architecture, only tested with implementation of hono js, drizzle orm, bun js, and postgres + sqlite. There may be variations in implementation depending on the framework and database used. Also Typescript and Zod for validation is strongly recommended or any other that can satisfy similar qualities of those two.

## License & Contributing

This architecture is still experimental and subject to change. So contributions, criticism and feedback are welcome.

Created by [Hussein Kizz](https://github.com/Hussseinkizz) at Nile Squad Labz. Completely open source under MIT License and used in production.


