
# The "REST-RPC" Architecture

This document provides a complete overview of the "REST-RPC" API architecture, a hybrid approach designed to streamline API interactions by centralizing operations while retaining key aspects of REST.

## 1. Introduction to "REST-RPC"

The "REST-RPC" architecture blends the resource-centric nature of REST with the operation-focused style of Remote Procedure Calls (RPC). It aims to simplify API design and client interaction by reducing the number of unique endpoints and explicitly defining actions through a standardized request body.

## 2. Core Principles

* **Resource-Based URLs:** The API is organized around logical resources, accessible via standard URLs (e.g., `/api/users`, `/api/products`). API versioning can be implemented as part of the base URL (e.g., `/api/v1/users`).
* **`GET` for State Retrieval and Discovery:**
    * Retrieves lists of resources (`GET /api/{resource}`). Responses may include metadata about available actions. Supports standard query parameters for pagination (`page`, `pageSize`) and filtering (`field=value`).
    * Retrieves specific resource details (`GET /api/{resource}/{resourceId}`).
* **`POST` for Action Invocation:** All operations that modify state or perform specific actions on a resource are typically handled via `POST` requests to the primary resource endpoint (`/api/{resource}`).
* **Standardized `POST` Request Body:** `POST` requests use a consistent JSON structure to specify the operation:
    ```json
    {
      "resourceId": "optional_resource_identifier",
      "action": "the_action_name",
      "payload": {
        // Data required for the action
      }
    }
    ```
* **Standardized Response Format:** All API responses adhere to a consistent JSON structure:
    ```json
    {
      "status": true|false,
      "message": "Descriptive message about the operation's outcome.",
      "data": { ... } // The actual data returned by the operation (if any).
    }
    ```
* **API Discovery:**
    * `GET /api/docs`: Lists all available API resources and their primary supported actions.
    * `GET /api/docs/{resource}`: Provides detailed information about the actions available for a specific resource, including descriptions and required/optional parameters in the `payload`.

## 3. Comparison with REST and RPC

| Feature             | REST (Traditional)                                    | RPC (Traditional)                                     | "REST-RPC"                                                                    |
| :------------------ | :---------------------------------------------------- | :---------------------------------------------------- | :---------------------------------------------------------------------------- |
| **Focus**           | Resources and their state                             | Operations/Procedures                                 | Hybrid: Resources are central, but actions are explicit.                      |
| **Endpoints**       | Multiple per resource (e.g., `/users`, `/users/{id}`) | Often a single endpoint or service-specific endpoints | Fewer endpoints per resource (primary `/api/{resource}`).                     |
| **Actions**         | Implicitly defined by HTTP methods                    | Explicitly defined in the request payload             | Explicitly defined by the `"action"` parameter in `POST` body.                |
| **HTTP Methods**    | Core reliance on GET, POST, PUT, DELETE               | Primarily POST (or custom methods)                    | GET for retrieval, POST for actions.                                          |
| **Discoverability** | Hypermedia (HATEOAS)                                  | Interface Definition Languages (IDLs)                 | Dedicated `/api/docs` endpoints and optional action hints in `GET` responses. |
| **Versioning**      | URI paths, headers                                    | Within procedure definitions, new service versions    | URI paths, implicit through action/payload evolution.                         |

## 4. Why Adopt "REST-RPC"?

* **Simplified Endpoint Management:** Reduces the complexity of routing and managing a large number of specific action endpoints.
* **Enhanced Action Clarity:** Explicitly names actions, making the intent of a request clearer.
* **Flexibility for Complex Operations:** Accommodates operations that don't fit neatly into standard CRUD semantics (e.g., `login`, `processOrder`).
* **Evolvable APIs:** Adding new actions or optional payload fields is less likely to break existing clients.
* **Centralized Action Handling:** Consolidates the processing logic for a resource's actions under a single endpoint handler.
* **Improved Discoverability (with `/docs`):** Provides clear, machine-readable documentation of available endpoints and their actions.

## 5. How it Works in Detail (Example: `/api/products`)

### 5.1. Listing Products

* **Request:** `GET /api/products?page=2&pageSize=20&category=electronics`
* **Response (Success):**
    ```json
    {
      "status": true,
      "message": "Successfully retrieved products.",
      "data": [
        { "id": "101", "name": "Laptop X", "price": 1200 },
        { "id": "102", "name": "Smartphone Y", "price": 800 },
        // ... more products
      ],
      "availableActions": ["createProduct", "getProduct", "updatePrice", "deleteProduct"]
    }
    ```

### 5.2. Getting a Specific Product

* **Request:** `GET /api/products/101`
* **Response (Success):**
    ```json
    {
      "status": true,
      "message": "Successfully retrieved product.",
      "data": { "id": "101", "name": "Laptop X", "price": 1200, "description": "..." }
    }
    ```

### 5.3. Creating a New Product

* **Request:**
    ```
    POST /api/products
    Content-Type: application/json

    {
      "action": "createProduct",
      "payload": {
        "name": "Tablet Z",
        "price": 300,
        "description": "..."
      }
    }
    ```
* **Response (Success):**
    ```json
    {
      "status": true,
      "message": "Product created successfully.",
      "data": { "id": "103", "name": "Tablet Z", "price": 300 }
    }
    ```

### 5.4. Updating a Product's Price

* **Request:**
    ```
    POST /api/products
    Content-Type: application/json

    {
      "resourceId": "101",
      "action": "updatePrice",
      "payload": {
        "newPrice": 1250
      }
    }
    ```
* **Response (Success):**
    ```json
    {
      "status": true,
      "message": "Product price updated successfully.",
      "data": { "id": "101", "name": "Laptop X", "price": 1250 }
    }
    ```

### 5.5. Getting Documentation for `/api/products`

* **Request:** `GET /api/docs/products`
* **Response (Success):**
    ```json
    {
      "resource": "products",
      "actions": {
        "createProduct": {
          "description": "Creates a new product.",
          "requiredPayload": ["name", "price"],
          "optionalPayload": ["description", "category"]
        },
        "getProduct": {
          "description": "Retrieves details of a specific product.",
          "requiredResourceId": true
        },
        "updatePrice": {
          "description": "Updates the price of an existing product.",
          "requiredResourceId": true,
          "requiredPayload": ["newPrice"]
        },
        "deleteProduct": {
          "description": "Deletes a specific product.",
          "requiredResourceId": true
        }
      }
    }
    ```

You are absolutely correct! Let's add that crucial point about achieving idempotency within the "Shortcomings and Considerations" section (which we can implicitly include within the "When to Use and When to Avoid" section for brevity and impact).

Here's the updated section:

## 6. When to Use and When to Avoid

**Use When:**

* Dealing with resources that have a wide range of operations beyond standard CRUD.
* Aiming for a more centralized and potentially simpler client-side interaction model.
* Prioritizing explicit action naming for better clarity.
* Building internal or closely coupled APIs where strict REST adherence is less critical.
* Desiring a degree of implicit versioning through action and payload evolution.
* Implementing robust API documentation is a priority.
* **Idempotency Requirements:** Idempotency can be achieved for `POST` requests by considering the combination of the `action` and `resourceId`. The server can track processed requests based on this combination to ensure that duplicate requests have the same effect as the original.

**Avoid When:**

* Building public-facing APIs where predictability and adherence to REST conventions are paramount for external developers.
* The resource model naturally aligns well with standard CRUD operations and HTTP method semantics.
* Heavily relying on the inherent caching and idempotency characteristics of specific HTTP methods (PUT, DELETE) without implementing explicit handling. While idempotency can be achieved as noted above, it requires explicit server-side logic.
* Integration with existing RESTful ecosystems is a primary concern and deviating might introduce friction.
* Hypermedia-driven discoverability (HATEOAS) is a key requirement.

**Note on Idempotency:** While the "REST-RPC" architecture uses `POST` for various actions, idempotency (the property that multiple identical requests have the same effect as one request) can still be implemented on the server-side. This can be achieved by uniquely identifying requests based on a combination of the `action` and the `resourceId` (if applicable). The server can then track processed requests and ensure that subsequent identical requests are handled without unintended side effects.

## 7. Conclusion

The "REST-RPC" architecture offers a pragmatic approach to API design, particularly for complex applications where a purely RESTful model might lead to an overwhelming number of endpoints. By centralizing actions under a consistent `POST` request structure and providing clear documentation, it aims to improve API usability and maintainability while still leveraging the fundamental concepts of resources and standard HTTP methods. Careful consideration of the trade-offs compared to traditional REST and RPC is essential to determine its suitability for a given project.
