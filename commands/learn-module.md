---
name: learn-module
description: Analyze a Drupal contrib/custom module and generate a skill reference from its API. Usage: /learn-module <module_name>
argument-hint: <module_name>
allowed-tools: [Read, Glob, Grep, Bash, Write, Agent]
model: sonnet
---

# Learn Module Command

Analyze a Drupal module's codebase and generate a skill reference document that captures its API surface, hooks, plugins, services, events, and configuration patterns.

## Arguments

The user provided: $ARGUMENTS

Parse the first argument as the module machine name (e.g., "paragraphs", "webform", "token", "pathauto").

## Workflow

### Step 1: Locate the Module

Search for the module in these directories (in order):
- `web/modules/contrib/$ARGUMENTS/`
- `modules/contrib/$ARGUMENTS/`
- `web/core/modules/$ARGUMENTS/`
- `core/modules/$ARGUMENTS/`
- `vendor/drupal/$ARGUMENTS/`

If the module is found, read its `.info.yml` file to confirm.

If not found, tell the user:
> Module "$ARGUMENTS" not found. Make sure the module is installed. Try:
> - `ddev composer require drupal/$ARGUMENTS`
> - Or provide the full path: `/learn-module /path/to/module`

### Step 2: Analyze the Module

Use the **module-learner** agent to perform a comprehensive analysis of the module. Pass the module name and its located path.

The agent will analyze:
- **Services** (`*.services.yml`) — public API services
- **Hooks provided** (`.api.php`, `moduleHandler()->invokeAll`) — hooks other modules can implement
- **Hooks implemented** (`.module`, `#[Hook]`) — what the module hooks into
- **Events** — custom event classes and dispatched events
- **Plugins** — plugin implementations and custom plugin types defined
- **Entities** — content and config entity types
- **Configuration** — config schema, install config, optional config
- **Permissions** — static and dynamic permissions
- **Routes** — admin and public routes

### Step 3: Generate Skill

Create the output directory:
```bash
mkdir -p .claude/skills/learned-modules/$ARGUMENTS
```

Write the analysis as a SKILL.md file to:
`.claude/skills/learned-modules/$ARGUMENTS/SKILL.md`

The file must include proper frontmatter:
```yaml
---
name: learned-{module_name}
description: API reference for the {module_name} Drupal module. Use when implementing features that integrate with or extend {module_name}.
version: 1.0.0
---
```

### Step 4: Report to User

Summarize what was learned:

```
## Module Learned: {module_name}

**Skill saved to:** `.claude/skills/learned-modules/{module_name}/SKILL.md`

### What was found:
- **Services:** {count} ({list top 3 most important})
- **Hooks provided:** {count} ({list top 3})
- **Events:** {count}
- **Plugin types:** {count}
- **Entities:** {count} ({list names})
- **Permissions:** {count}

### How to use:
This skill will be automatically available in future conversations within
this project. When you ask Claude to implement features involving {module_name},
it will understand the module's API and use the correct integration patterns.

### Common integration patterns:
1. {Most common pattern, e.g., "Implement hook_X to alter behavior"}
2. {Second pattern, e.g., "Subscribe to X event for custom logic"}
3. {Third pattern, e.g., "Create a custom X plugin"}
```
