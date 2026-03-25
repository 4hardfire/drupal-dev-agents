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

After approval, implement the feature following the architecture:

1. **Module scaffold** — `.info.yml`, `.module`, `.services.yml`, `.routing.yml`
2. **Configuration** — schema, default config, settings form
3. **Entities** — entity classes, base fields, handlers
4. **Services** — business logic classes, service definitions
5. **Plugins** — block, field, action, etc.
6. **Forms** — config forms, entity forms, custom forms
7. **Controllers** — page controllers, route handlers
8. **Hooks/Events** — hook implementations, event subscribers
9. **Permissions** — permissions.yml, access handlers
10. **Templates** — Twig templates, preprocess functions, libraries

Use `ddev drush gen` where appropriate to scaffold boilerplate.

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
