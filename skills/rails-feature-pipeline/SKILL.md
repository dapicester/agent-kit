---
name: rails-feature-pipeline
description: End-to-end implementation of a feature in a Rails project — verify the required model attributes exist and migrate with `rails generate migration` when they don't, write tests (integration for API/complex features, unit for simple model changes), update the OpenAPI spec for every endpoint change, run the full test suite, then have a tester agent review the implementation. Use when the user asks to implement, add, or change a feature, endpoint, model attribute, or migration.
---

# Feature Pipeline

Implement a Rails feature end-to-end, in order. Each step ends when its completion criterion is met; do not move past a step whose criterion is unmet.

## 1. Attributes exist → migrate if not

For every entity the feature touches, verify the required columns actually exist. Check `db/schema.rb` or the model file; if the project exposes a ground-truth tool (e.g. rails-ai-context MCP), query that instead of guessing.

Missing a column? Create it with the official generator, never by hand — it stamps versions and updates `schema.rb` correctly:

```sh
bin/rails generate migration AddTimeZoneToUsers time_zone:string
bin/rails db:migrate
bin/rails db:migrate:status   # new migration shows up
```

**Criterion:** `db/schema.rb` contains every required column and the migration is `up`.

## 2. RESTful controllers and actions

A new endpoint gets its own controller named for the resource it acts on, with a standard action name (`index`, `show`, `create`, `update`, `destroy`). Do not bolt a custom action onto an existing controller (`V1::UsersController#credits`) or invent a verb-shaped action (`#reset_password` on a passwords controller that already has `create`); mount the resource and use `create`.

```ruby
# config/routes.rb — nested under an existing resource, controller named explicitly
resource :credit_charge, only: :create, controller: "users_credit_charges"

# app/controllers/v1/users_credit_charges_controller.rb
class V1::UsersCreditChargesController < V1::ApplicationController
  def create
    params.expect(credit_charge: [ :credits ])
  end
end
```

Pick the route form by what the client does with the resource afterwards:

- **Only ever created** (`only: :create`) → singular `resource :credit_charge`. Rails' default
  pairing is a singular route key with a plural controller (`resource :rating` →
  `RatingsController`, `resource :spending` → `SpendingsController`). The payload wrapper is the
  singular route key: `{ "credit_charge": { "credits": 250 } }`.
- **Listed or addressed by id later** → plural `resources :meals`, and use `:member`/`:collection`
  options instead of inventing verb actions.

Success returns the representation of what the action produced, not just a status.

**Criterion:** every new endpoint maps to a resource-named controller and a standard action name.

### Web view styling

Page-specific styles go in `app/assets/stylesheets/<page>.css`, never in inline `<style>` partials rendered from views. Propshaft bundles every CSS file there site-wide (`stylesheet_link_tag :app`), so scope rules that would leak: namespace classes (`.yp-*`), and scope bare element selectors like `body` with `body:has(.page-marker-class)`.

## 3. Tests by feature shape

- **API endpoint or multi-layer feature** → integration test under `test/integration/`. Cover the three cases: auth (401 without credentials), happy path (write then read back what you wrote), error path (expected errors hash, 4xx/422).
- **Simple model or validation change** → unit test under `test/models/`.
- Assert literal expected values in response bodies rather than deriving them from fixtures.

**Criterion:** tests for the changed behavior exist and pass.

## 4. OpenAPI spec on every endpoint change

Any endpoint added, changed, or removed → update the project's OpenAPI spec (commonly `docs/openapi.yml`) in the same change, keeping request body and response schema consistent. Match the file's existing YAML style and verify it parses before moving on.

**Criterion:** spec parses (e.g. `ruby -e 'require "yaml"; YAML.load_file("docs/openapi.yml")'`) and describes the new contract.

## 5. Full suite green

Run the touched test files first, then the whole suite (`bin/rails test`). Leave it at zero failures and zero errors.

**Criterion:** full suite passes.

## 6. Tester agent review

Dispatch a tester subagent to review the implementation: run the relevant tests, read the diff, and check the code against the original request. Investigator-only — the tester reviews, it does not edit. Fold real findings into fixes and re-run the suite; acknowledge anything intentionally left.

**Criterion:** review delivered; findings addressed or explicitly acknowledged to the user.

## 7. Update Rails AI context

Update the AI context used by the Rails MCP, so the next session sees the new models, routes, and controllers instead of a stale snapshot:

```sh
bin/rails ai:context
```

**Criterion:** the command completes and the regenerated context files (e.g. `AGENTS.md`, `CLAUDE.md`, `.claude/rules/`) include the models, controllers, and routes touched by this change.
