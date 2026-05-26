# tenancy-development

An [Agent Skill](https://agentskills.io) for developing multi-tenant Laravel applications with [stancl/tenancy](https://tenancyforlaravel.com) v3.

This skill equips AI coding agents with comprehensive knowledge of multi-tenant architecture — from tenant identification, bootstrappers, and the event system to production patterns for integrating Spatie, Scout, Livewire, and Horizon.

## What It Covers

- **Package architecture**: central vs tenant contexts, automatic/manual modes
- **Tenant identification**: domain, subdomain, path, and request-data middleware
- **Bootstrappers**: database, cache, filesystem, queue, and Redis scoping
- **Custom bootstrappers**: Scout/Meilisearch index scoping with prefix accumulation guards
- **Features**: TenantConfig for per-tenant Laravel config, UserImpersonation, TelescopeTags
- **Event system**: full lifecycle (creating, created, deleting, deleted) with JobPipeline
- **Production patterns**: real-world TenantServiceProvider, route middleware stacks, Horizon job tagging
- **Third-party integration**: Spatie Permission, Spatie MediaLibrary, Livewire, Horizon, queues

## Installation

### Claude Code

Copy the SKILL.md to your skills directory:

```bash
mkdir -p ~/.claude/skills/tenancy-development
curl -o ~/.claude/skills/tenancy-development/SKILL.md https://raw.githubusercontent.com/Xyntax01/tenancy-development/main/SKILL.md
```

Or clone the repo:

```bash
git clone https://github.com/Xyntax01/tenancy-development ~/.claude/skills/tenancy-development
```

### OpenCode

Copy the SKILL.md to your agents skills directory:

```bash
mkdir -p .agents/skills/tenancy-development
curl -o .agents/skills/tenancy-development/SKILL.md https://raw.githubusercontent.com/Xyntax01/tenancy-development/main/SKILL.md
```

### Codex CLI

```bash
mkdir -p ~/.codex/skills/tenancy-development
curl -o ~/.codex/skills/tenancy-development/SKILL.md https://raw.githubusercontent.com/Xyntax01/tenancy-development/main/SKILL.md
```

### Other agents

Any agent that supports the [Agent Skills specification](https://agentskills.io/specification) can use this skill. Place the `SKILL.md` in the appropriate skills directory for your agent.

## Quick Install

```bash
git clone https://github.com/Xyntax01/tenancy-development.git && \
  cp -r tenancy-development/SKILL.md ~/.claude/skills/tenancy-development/SKILL.md && \
  echo "Skill installed!"
```

## License

MIT
