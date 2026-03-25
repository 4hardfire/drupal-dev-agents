# Drupal Dev Agents

A comprehensive Claude Code plugin for Drupal 10/11+ development. Provides 17 specialized skills, 4 intelligent agents, and 4 slash commands covering the entire Drupal development workflow.

## Installation

Add this repository as a custom marketplace in your Claude Code settings:

```json
{
  "extraKnownMarketplaces": {
    "drupal-dev-agents": {
      "source": {
        "source": "github",
        "repo": "4hardfire/drupal-dev-agents"
      }
    }
  }
}
```

Then install the plugin:

```
/plugin install drupal-dev-agents@drupal-dev-agents
```

## Skills (17)

### Drupal API Skills
| Skill | Description |
|-------|-------------|
| **drupal-plugins** | Plugin system — Block, Field, Action, QueueWorker, Condition (D10 annotations + D11+ attributes) |
| **drupal-entities** | Content entities and config entities — base fields, handlers, forms, list builders |
| **drupal-forms** | Forms API — ConfigFormBase, FormBase, multi-step, AJAX forms |
| **drupal-services** | Services and DI — service definitions, constructor injection, tagged services, decorators |
| **drupal-events** | Event system — subscribers, custom events, kernel/entity/config events |
| **drupal-hooks** | Hooks — legacy procedural AND Drupal 11+ OOP hooks with `#[Hook]` attribute |
| **drupal-routing** | Routing — YAML routes, controllers, route subscribers, parameter upcasting |
| **drupal-database** | Database — schema, update hooks, post_update, Database API queries |
| **drupal-render** | Render system — render arrays, elements, hook_theme(), Twig, lazy builders |
| **drupal-access** | Access control — permissions, access checkers, entity access handlers |
| **drupal-queue** | Queue API — QueueWorker plugins, cron processing, batch operations |
| **drupal-cache** | Cache API — tags, contexts, max-age, bins, invalidation |
| **drupal-config** | Configuration — simple config, config entities, schema, overrides |
| **drupal-testing** | Testing — Unit, Kernel, Functional, Browser tests with PHPUnit |

### Tooling Skills
| Skill | Description |
|-------|-------------|
| **drush-commands** | All operational Drush commands — cache, config, deploy, database, user, cron, watchdog, queues, maintenance mode |
| **drush-generate** | All `drush gen` commands with options and non-interactive usage |
| **ddev-setup** | DDEV project setup, configuration, services, xdebug |

## Agents (4)

| Agent | Description |
|-------|-------------|
| **module-learner** | Analyzes any contrib/core module and generates a skill from its API |
| **drupal-code-explorer** | Drupal-aware codebase exploration |
| **drupal-code-reviewer** | Drupal coding standards, security, and performance review |
| **drupal-architect** | Module architecture design and decision-making |

## Commands (4)

| Command | Usage | Description |
|---------|-------|-------------|
| `/learn-module` | `/learn-module paragraphs` | Analyze a module and generate a skill reference |
| `/drupal-feature` | `/drupal-feature "donation tracker"` | Guided feature development workflow |
| `/ddev-init` | `/ddev-init 11` | Interactive DDEV + Drupal setup |
| `/drush-gen` | `/drush-gen plugin:block` | Guided drush code generation |

## Module Learner

The module learner is the standout feature. Point it at any Drupal module:

```
/learn-module paragraphs
```

It will analyze the module's services, hooks, events, plugins, entities, configuration, and permissions, then generate a project-local skill at `.claude/skills/learned-modules/paragraphs/SKILL.md`. Future conversations will automatically understand that module's API.

## Requirements

- Claude Code CLI
- Drupal 10 or 11+ project
- DDEV (for tooling skills)
- Drush (for code generation skills)
