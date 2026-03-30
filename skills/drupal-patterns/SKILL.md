---
name: drupal-patterns
description: Use when deciding which Drupal pattern to use for a feature — content entity vs config entity vs plugin vs service vs event subscriber vs hook vs queue worker vs batch. Decision framework only, no code samples.
version: 1.0.0
---

# Drupal Pattern Selection Guide

## When to Use Each Pattern

### Content Entity

Use when you need:
- Fieldable, structured data with CRUD operations
- Admin UI with list, add, edit, delete pages
- Views integration for listing, filtering, sorting
- Per-entity access control
- Revisions, translations, or workflow states
- Relationships to other entities via entity reference fields
- User-generated or runtime data storage

**Examples:** Nodes, Users, Comments, Media, custom records (tasks, orders, tickets).

**Drush generator:** `ddev drush gen entity:content`

---

### Config Entity

Use when you need:
- Admin-created configuration objects (like image styles, views, text formats)
- Exportable/importable via config management (`drush cex` / `drush cim`)
- A set of named, configurable "things" (not user content)
- Machine name + label pattern
- No field UI (properties defined in code)
- Bundle providers for content entities

**Examples:** View, ImageStyle, Role, Vocabulary, NodeType, custom type definitions.

**Drush generator:** `ddev drush gen entity:config`

---

### Simple Configuration

Use when you need:
- A single settings object for a module (API keys, feature flags, display preferences)
- Admin settings form (`ConfigFormBase`)
- Values that should be exported with config management

**Not for:** Multiple instances of the same structure (use config entity instead).

**Drush generator:** `ddev drush gen form:config` (creates form + schema + route)

---

### Plugin

Use when you need:
- Swappable implementations (choose one from a list)
- Admin-selectable behavior (which block, which field widget)
- Discovery-based extensibility (other modules add their own implementations)

**Use existing plugin types first:** Block, FieldType, FieldWidget, FieldFormatter, Action, QueueWorker, Condition, Constraint, Views handlers.

**Drush generators:**
- `ddev drush gen plugin:block`
- `ddev drush gen plugin:field:type`
- `ddev drush gen plugin:field:widget`
- `ddev drush gen plugin:field:formatter`
- `ddev drush gen plugin:queue-worker`
- `ddev drush gen plugin:action`
- `ddev drush gen plugin:condition`

---

### Custom Plugin Type

Use **only** when:
- No existing plugin type fits your needs
- You need a discoverable set of interchangeable implementations
- Multiple modules should be able to provide implementations

**No drush generator** -- must be written manually (attribute class, interface, plugin manager, service definition).

---

### Service

Use when you need:
- Reusable business logic (the most common pattern)
- Dependency injection
- Stateless operations
- Overridable/decoratable behavior
- A singleton that other code can depend on

**Drush generator:** `ddev drush gen service`

---

### Event Subscriber

Use when you need:
- React to kernel-level operations (request, response, exceptions)
- React to configuration changes
- React to entity lifecycle operations (Drupal 10.3+)
- Dispatch your own decoupled notifications
- Priority-based ordering of listeners
- Loose coupling between modules

**Drush generator:** `ddev drush gen event-subscriber`

---

### Hook (OOP in D11+)

Use when you need:
- Alter existing behavior (form_alter, entity_view_alter, theme suggestions)
- Implement a contract defined by another module
- Module/theme layer integration
- Respond to entity CRUD operations

**Drush generator:** `ddev drush gen hook`

---

### Queue Worker

Use when you need:
- Deferred/background processing
- Long-running operations outside the HTTP request
- Retry-able, fault-tolerant tasks
- Cron-triggered processing

**Drush generator:** `ddev drush gen plugin:queue-worker`

---

### Batch Operation

Use when you need:
- Process large datasets with a progress bar
- User-initiated bulk operations
- Operations that might time out in a single request

**No drush generator** -- must be written manually (batch_set, operation callback, finished callback).

---

## Quick Decision Matrix

| Feature need | Pattern |
|---|---|
| Users create/manage structured data | Content entity + entity form + list builder + views + permissions |
| Admins configure reusable settings objects | Config entity + entity form + list builder + config schema |
| Module needs a settings page | Simple config + ConfigFormBase + route + permissions |
| System needs to react to things happening | Event subscriber (preferred) or hook implementation |
| Content displayed in configurable ways | Block plugin, FieldFormatter plugin, or Views display plugin |
| Data needs processing in the background | Queue + QueueWorker plugin + cron or manual trigger |
| External API integration | Service (HTTP client wrapper) + config (credentials) + queue (async ops) |
| Alter existing forms/entities/render | Hook (form_alter, entity_presave, preprocess) |
| Scheduled cleanup or sync | hook_cron or QueueWorker with cron |
| Bulk user-initiated operation | Batch API with progress bar |
| Large background processing | Batch API to enqueue + QueueWorker to process via cron |

---

## Events vs Hooks Decision

| Use Events When | Use Hooks When |
|---|---|
| Reacting to kernel-level operations | Altering forms (`hook_form_alter`) |
| Subscribing to configuration changes | Defining theme implementations (`hook_theme`) |
| Reacting to entity lifecycle (Drupal 10.3+) | Providing data to Views (`hook_views_data`) |
| Dispatching your own decoupled notifications | Altering existing plugin definitions |
| Needing priority-based ordering | Working with legacy module APIs |

---

## Content Entity vs Config Entity

| Aspect | Content Entity | Config Entity |
|---|---|---|
| Storage | Database tables | YAML config files |
| Revisionable | Yes (optional) | No |
| Translatable | Yes (optional) | Yes (limited) |
| Fieldable | Yes (via Field UI) | No |
| Exportable | Only via migrate / default content | Yes (config export) |
| Use when | Storing user-generated or runtime data | Storing site-builder settings, reusable config objects |

---

## Database API vs Entity API

| Use Database API | Use Entity API |
|---|---|
| Custom tables not backed by entities | Loading, saving, deleting entities |
| Aggregation and reporting queries | CRUD on nodes, users, taxonomy terms |
| Performance-critical bulk operations | Anything needing hooks, access control, validation |
| Legacy tables or external data sources | Field data on fieldable entities |
| Logging, statistics, session tables | Content and configuration entities |

---

## Related Skills

- **drush-generate** -- Drush code generation commands for scaffolding all patterns.
