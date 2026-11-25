---
name: openapi-updater
description: Updates the OpenAPI specification document based on existing routes and test cases.
---

# OpenAPI Updater

A specialized skill for maintaining the `docs/openapi.yml` specification in synchronization with the Rails application code.

## Objective
Ensure `docs/openapi.yml` accurately reflects the current state of the API, including all routes, request parameters, response structures, and component schemas.

## Workflow

### 1. Discovery (Analyze Routes & Controllers)
- **Scan Routes:** Inspect `config/routes.rb` for all routes under the `namespace :v1` block. Identify new, modified, or deleted endpoints.
- **Inspect Controllers:** For any changed endpoint, read the corresponding controller action in `app/controllers/v1/`.
    - Determine the HTTP method (GET, POST, PUT, DELETE, etc.).
    - Identify parameters extracted from `params` (query, path, or body).
    - Determine the success response structure and any potential error responses (401, 404, 422, etc.).

### 2. Model Introspection (Update Schemas)
- **Fetch Model Context:** If an endpoint interacts with a model (e.g., `Creature`, `Meal`), use `rails_get_context(model: "ModelName")` or `rails_analyze_feature` to retrieve the current schema and associations.
- **Sync Components:** Update `components/schemas` in `docs/openapi.yml` if the model's attributes, enums, or nested structures have changed.

### 3. Specification Update (The Implementation)
- **Edit Paths:** Add or update entries in the `paths:` section of `docs/openapi.yml`.
- **Define Operation Details:**
    - `summary`: A concise description of the action.
    - `operationId`: A camelCase identifier.
    - `tags`: Categorize endpoints (e.g., `Authentication`, `User`, `Meals`, `Restaurants`).
    - `security`: Include `bearerAuth: []` where authentication is required.
- **Define Request/Response Bodies:**
    - Map `application/json` schemas for POST/PUT/PATCH requests.
    - Map request bodies for `multipart/form-data` when applicable (e.g., file uploads).
    - Ensure all responses (2xx, 4xx, 5xx) are documented with clear schemas or error messages.
- **Reference Schemas:** Use `$ref: '#/components/schemas/Name'` for reusable objects to keep the specification DRY.

### 4. Verification
- **Validate YAML:** Ensure the update results in valid YAML.
- **Consistency Check:** Verify that all `operationId`s are unique and that paths match the actual `config/routes.rb` definitions.

## Rules
- **DRY (Don't Repeat Yourself):** Always prefer `components/schemas` over inline object definitions.
- **Minimal Diff:** Only update parts of the specification that have changed.
- **Precision:** Ensure all data types (integer, string, boolean, etc.) and formats (email, float, date-time) are correctly specified.
- **Tagging:** Every new path must have at least one tag assigned.
