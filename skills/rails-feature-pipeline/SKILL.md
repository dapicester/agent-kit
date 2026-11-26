---
name: rails-feature-pipeline
description: "End-to-end implementation of a feature in a Rails project: verify the required model attributes exist and migrate with `rails generate migration` when they don't, write tests (integration for API and multi-layer features, unit for simple model changes), update the OpenAPI spec for every endpoint change, run the full test suite, lint with RuboCop when the project has it, have a reviewer subagent check the implementation, and refresh the AI context snapshot (`rails ai:context`) when the project generates one. Use when the user asks to implement, add, or change a feature, endpoint, model attribute, or migration."
---

# Rails Feature Pipeline

Implement a Rails feature end-to-end, in order. Each step ends when its completion criterion is met; do not move past a step whose criterion is unmet.

## 1. Attributes exist, migrate if not

For every entity the feature touches, verify the required columns actually exist. Check `db/schema.rb` or the model file; if the project exposes a ground-truth tool (a Rails context MCP, a generated AI context, anything that reads the live schema), query that instead of guessing.

Missing a column? Create it with the official generator, never by hand, so the version stamp and `schema.rb` stay correct:

```sh
bin/rails generate migration AddStatusToWidgets status:string
bin/rails db:migrate
bin/rails db:migrate:status   # new migration shows up
```

**Criterion:** `db/schema.rb` contains every required column and the migration is `up`.

## 2. RESTful controllers and actions

A new endpoint gets its own controller named for the resource it acts on, with a standard action name (`index`, `show`, `create`, `update`, `destroy`). Do not bolt a custom action onto an existing controller (`CommentsController#approve`) or invent a verb-shaped action (`#reset_password` on a passwords controller that already has `create`); mount the resource and use `create` instead.

```ruby
# config/routes.rb: controller named explicitly
resource :charge, only: :create, controller: "charges"

# app/controllers/charges_controller.rb
class ChargesController < ApplicationController
  def create
    params.expect(charge: [ :amount ])
  end
end
```

In an API-only app, inherit from the project's API base controller and keep its namespace and versioning convention (`V1::BaseController` or whatever the app already uses) rather than introducing a new one.

Pick the route form by what the client does with the resource afterwards:

- **Only ever created** (`only: :create`) → singular `resource :charge`. Rails' default
  pairing is a singular route key with a plural controller (`resource :rating` →
  `RatingsController`, `resource :spending` → `SpendingsController`). The payload wrapper is the
  singular route key: `{ "charge": { "amount": 250 } }`.
- **Listed or addressed by id later** → plural `resources :meals`, and use `:member`/`:collection`
  options instead of inventing verb actions.

Success returns the representation of what the action produced, not just a status.

**Criterion:** every new endpoint maps to a resource-named controller and a standard action name.

### Web view styling

Page-specific styles go in `app/assets/stylesheets/<page>.css`, never in inline `<style>` partials rendered from views. Rails bundles every CSS file there site-wide (`stylesheet_link_tag :app` with Propshaft or Sprockets), so scope rules that would leak: namespace classes (`.<page>-*`) and scope bare element selectors with a page marker, e.g. `body:has(.<page>-marker)`.

## 3. Tests by feature shape

- **API endpoint or multi-layer feature** → integration test under `test/integration/`. Cover the three cases: auth (401 without credentials), happy path (write then read back what you wrote), error path (expected errors hash, 4xx/422).
- **Simple model or validation change** → unit test under `test/models/`.
- Assert literal expected values in response bodies rather than deriving them from fixtures.
- Use the project's existing test framework and fixtures as they are. Do not introduce a new one.

**Criterion:** tests for the changed behavior exist and pass.

## 4. OpenAPI spec on every endpoint change

Any endpoint added, changed, or removed → update the project's OpenAPI spec (commonly `docs/openapi.yml`) in the same change, keeping request body and response schema consistent. Match the file's existing YAML style and verify it parses before moving on.

**Criterion:** spec parses (e.g. `ruby -e 'require "yaml"; YAML.load_file("docs/openapi.yml")'`) and describes the new contract.

## 5. Lint and full suite green

If the project has RuboCop installed (a `rubocop` or `rubocop-rails` entry in the Gemfile, a `.rubocop.yml`, or a `bin/rubocop` binstub), lint the files you touched and fix what it reports:

```sh
bin/rubocop app/controllers/charges_controller.rb test/integration/charges_test.rb
```

No binstub? Use `bundle exec rubocop`. If the project lints the whole tree before committing, run the bare `bin/rubocop` too. Fix offenses in code, never by disabling cops or widening the excludes; if a cop is genuinely wrong for this code, leave the config alone and say so in the summary. Skip this when no linter is present, and do not add RuboCop to a project that does not have it.

Then run the touched test files first, then the whole suite (`bin/rails test` or the project's runner). Leave it at zero failures, zero errors, and no new lint offenses.

**Criterion:** full suite passes and the touched files are lint-clean (or the project has no linter).

## 6. Reviewer subagent

Dispatch a reviewer subagent to check the implementation: run the relevant tests, read the diff, and compare the code against the original request. Investigator only, the reviewer does not edit. Fold real findings into fixes and re-run the suite; acknowledge anything intentionally left. If the harness has no subagent support, do this review inline before claiming the task done.

**Criterion:** review delivered; findings addressed or explicitly acknowledged to the user.

## 7. Update the AI context, if the project has one

If the project regenerates an AI context snapshot (a rake task such as `bin/rails ai:context`, a `rails-ai-context` install, any generated `AGENTS.md` / `CLAUDE.md` / rules directory), run it so the next session sees the new models, routes, and controllers instead of a stale snapshot, and commit the regenerated files together with (or right after) the feature — a snapshot only useful on your machine helps no one. Verify the generator touched only its managed marker blocks, never the hand-written sections of those files. Skip this step when no such task exists; do not invent one.

**Criterion:** either the command completes, the regenerated files mention the models, controllers, and routes touched by this change and are committed, or the step is skipped with a one-line note saying the project has no context task.
