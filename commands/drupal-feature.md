---
name: drupal-feature
description: Guided Drupal feature development workflow — explores, architects, implements, and reviews. Usage: /drupal-feature <description>
argument-hint: <feature description>
allowed-tools: [Read, Glob, Grep, Bash, Write, Edit, Agent]
---

# Drupal Feature Development Command

Guide the implementation of a Drupal feature through a structured, multi-phase workflow.

## Arguments

The user provided: $ARGUMENTS

This describes the feature they want to build.

## Workflow

### Phase 1: Discovery

Understand the feature requirements from $ARGUMENTS. Ask clarifying questions if needed:

- **Drupal version?** Check `core/lib/Drupal.php` or `composer.json` for `drupal/core` version
- **Target module?** New custom module or extend an existing one?
- **Contrib modules?** Are there existing contrib modules that solve part of this?
- **Audience?** Who uses this feature — admins, content editors, end users?
- **Data model?** What data needs to be stored and how?

### Phase 2: Exploration

Use the **drupal-code-explorer** agent to:
- Explore the current codebase for related functionality
- Check installed contrib modules that might help
- Identify existing patterns and conventions in the project
- Find similar implementations to follow as examples

### Phase 3: Architecture

Use the **drupal-architect** agent to design:
- Module structure (new or extending existing)
- Entity types (if data storage needed)
- Services (business logic)
- Plugins (blocks, fields, etc.)
- Routes, forms, and controllers
- Permissions and access control
- Configuration schema
- Integration points (hooks, events)
- Testing strategy

**Present the architecture to the user for approval before proceeding to implementation.**

### Phase 4: Implementation

After approval, implement the feature following the architecture. **Always use `ddev drush gen` as the primary tool for generating boilerplate code.** Only write boilerplate manually when no drush generator exists for that specific component — and even then, follow Drupal coding standards and conventions exactly as drush would generate them.

#### Step 1: Generate all boilerplate with drush

Run the appropriate `ddev drush gen` commands in sequence. Use `--answers` JSON for non-interactive mode. The mapping is:

| Component | Generator |
|-----------|-----------|
| Module scaffold | `ddev drush gen module` |
| Content entity | `ddev drush gen entity:content` |
| Config entity | `ddev drush gen entity:config` |
| Controller + route | `ddev drush gen controller` |
| Config form | `ddev drush gen form:config` |
| Simple form | `ddev drush gen form:simple` |
| Service | `ddev drush gen service` |
| Event subscriber | `ddev drush gen event-subscriber` |
| Block plugin | `ddev drush gen plugin:block` |
| Field type plugin | `ddev drush gen plugin:field:type` |
| Field widget plugin | `ddev drush gen plugin:field:widget` |
| Field formatter plugin | `ddev drush gen plugin:field:formatter` |
| Queue worker plugin | `ddev drush gen plugin:queue-worker` |
| Action plugin | `ddev drush gen plugin:action` |
| Condition plugin | `ddev drush gen plugin:condition` |
| Hook implementation | `ddev drush gen hook` |
| Permissions YAML | `ddev drush gen yml:module:permissions` |
| Routing YAML | `ddev drush gen yml:module:routing` |
| Services YAML | `ddev drush gen yml:module:services` |
| Menu links YAML | `ddev drush gen yml:module:links:menu` |
| Task links YAML | `ddev drush gen yml:module:links:task` |
| Action links YAML | `ddev drush gen yml:module:links:action` |
| Tests | `ddev drush gen test` |
| Theme | `ddev drush gen theme` |

Run `ddev drush cr` after generating all boilerplate.

#### Step 2: Implement custom logic on top of the generated boilerplate

After drush has generated all scaffold files, edit them to add the feature-specific logic:

1. **Entity base fields** — Add/modify `baseFieldDefinitions()` with the required fields
2. **Service methods** — Implement business logic methods in the generated service classes
3. **Plugin logic** — Fill in `build()`, `blockForm()`, `processItem()`, etc.
4. **Form elements** — Add form fields, validation, and submission logic
5. **Controller responses** — Implement route handler logic
6. **Hook/Event logic** — Add the specific behavior in generated hook/subscriber methods
7. **Access logic** — Implement access checks in generated access handlers
8. **Config schema** — Define schema for any configuration the module stores
9. **Default config** — Create `config/install/` YAML files as needed
10. **Templates** — Create Twig templates, preprocess functions, and library definitions

#### Step 3: Handle components without a drush generator

For components that drush cannot generate (e.g., custom plugin types, Twig templates, preprocess functions, library definitions, batch operations, custom event classes, config schema files, default config YAML), write the code manually following:

- PSR-12 and Drupal coding standards
- Proper file docblocks and class docblocks
- Correct PSR-4 namespacing under `src/`
- Dependency injection patterns consistent with the generated code
- The same conventions and style as drush-generated files in the module

### Phase 5: Review

Use the **drupal-code-reviewer** agent to review for:
- Security vulnerabilities (XSS, SQL injection, access bypass)
- Performance issues (N+1 queries, missing cache metadata)
- Coding standards compliance
- Proper API usage (DI, entity API, render API)

Fix any critical issues found.

### Phase 6: Testing

Generate test scaffolding:
- **Unit tests** for service logic
- **Kernel tests** for entity CRUD, service integration
- **Functional tests** for UI workflows, access control

### Phase 7: Summary

Report what was created:

```
## Feature Complete: {feature_name}

### Files created/modified:
{list of files}

### To enable:
ddev drush en {module_name} -y
ddev drush cr

### To test:
ddev exec phpunit web/modules/custom/{module_name}/tests/

### Configuration:
- Settings: /admin/config/{category}/{module}
- Permissions: /admin/people/permissions#{module}

### Next steps:
{recommendations for further development}
```
