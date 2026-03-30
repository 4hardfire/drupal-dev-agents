# Drupal Dev Agents

A Claude Code plugin for Drupal 10/11+ development. Provides 4 focused skills, 4 intelligent agents, and 4 slash commands with a **drush-generate-first workflow** — all boilerplate is scaffolded with `ddev drush gen`, then custom logic is implemented on top.

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

## Philosophy

**Drush generate first, custom logic second.**

1. All boilerplate code is generated using `ddev drush gen` (modules, entities, plugins, forms, controllers, services, event subscribers, hooks, tests)
2. Custom logic is implemented on top of the generated scaffold
3. Only when drush has no generator for a component is the boilerplate written manually — still following Drupal/PHP standards

## Skills (4)

| Skill | Description |
|-------|-------------|
| **drupal-patterns** | Decision framework — when to use content entity vs config entity vs plugin vs service vs event vs hook vs queue |
| **drush-generate** | All `ddev drush gen` commands with options and non-interactive `--answers` JSON usage |
| **drush-commands** | Operational Drush commands — cache, config, deploy, database, user, cron, watchdog, queues |
| **ddev-setup** | DDEV project setup, configuration, services, xdebug |

## Agents (4)

| Agent | Description |
|-------|-------------|
| **module-learner** | Analyzes any contrib/core module and generates a skill from its API |
| **drupal-code-explorer** | Drupal-aware codebase exploration |
| **drupal-code-reviewer** | Drupal coding standards, security, and performance review |
| **drupal-architect** | Module architecture design with drush generator mapping |

## Commands (4)

| Command | Usage | Description |
|---------|-------|-------------|
| `/learn-module` | `/learn-module paragraphs` | Analyze a module and generate a skill reference |
| `/drupal-feature` | `/drupal-feature "donation tracker"` | Guided feature development workflow (drush-first) |
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
