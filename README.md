# ed3d-plugins-hailz

Hailey's personal fork of [`ed3d-plugins`](https://github.com/ed3dai/ed3d-plugins) — Ed's collection of Claude Code plugins for design, implementation, and development workflows. This fork carries personal modifications, additions, and tweaks I've made for my own day-to-day; it diverges from upstream where my workflow needs something different.

The big stick is `ed3d-plan-and-execute`, which implements an "RPI" (research-plan-implement) loop that does a really good job of avoiding hallucination in the planning stages, adhering to high-level product requirements, avoiding drift between design planning and implementation planning, and reviewing the results such that you get out the other end not just what you asked for, but what you actually wanted.

**NOTE:** This is *my* fork. If you want the upstream (more stable, more widely tested), see [`ed3dai/ed3d-plugins`](https://github.com/ed3dai/ed3d-plugins).

## Using `ed3d-plan-and-execute`
More in [the README for the plugin](plugins/ed3d-plan-and-execute/README.md), and it's worth skimming, but here's a quickstart:

```
Rough Idea
    │
    ▼
/start-design-plan  ──────► Design Document (committed to git)
    │
    ▼
/start-implementation-plan ──► Implementation Plan (phase files)
    │
    ▼
/execute-implementation-plan ──► Working Code (reviewed & committed)
```

**Customization:** Create `.ed3d/design-plan-guidance.md` and `.ed3d/implementation-plan-guidance.md` in your project to provide project-specific constraints, terminology, and standards. Run `/how-to-customize` for details.

## Plugins

| Plugin | Description |
|--------|-------------|
| **`ed3d-00-getting-started`** | Getting started guide and onboarding for ed3d-plugins. Run `/getting-started` to see this README. |
| **`ed3d-plan-and-execute`** | Planning and execution workflows for Claude Code. Feed it a decent-sized task and it'll help you get it done in a sustainable and thought-through way |
| **`ed3d-house-style`** | House style for software development; Very Opinionated |
| **`ed3d-basic-agents`** | Core agents for general-purpose tasks (haiku, sonnet, opus). Other plugins expect this to exist |
| **`ed3d-research-agents`** | Agents for research across multiple data sources (codebase, internet, combined); other plugins expect this to exist |
| **`ed3d-extending-claude`** | Knowledge skills for extending Claude Code: plugins, commands, agents, skills, hooks, MCP servers. Other plugins expect this to exist |
| **`ed3d-playwright`**| Playwright automation with subagents |
| **`ed3d-hook-skill-reinforcement`** | UserPromptSubmit hook that reinforces the need to activate skills—helps make sure skills actually get used. Requires `ed3d-extending-claude` to work |
| **`ed3d-hook-claudemd-reminder`** | PostToolUse hook that reminds to update CLAUDE.md before committing |
| **`ed3d-hook-security-hardening`** | PreToolUse and PostToolUse hooks that catch secrets leakage patterns |
| **`ed3d-session-reflection`** | EXPERIMENTAL. Session awareness and conversation review tooling. Requires `ed3d-extending-claude` |

## Installation

### Add the marketplace
```bash
/plugin marketplace add https://github.com/haileyok/ed3d-plugins-hailz.git
```

### Install plugins
All plugins are available from the `ed3d-plugins-hailz` marketplace:
```bash
/plugin install ed3d-plan-and-execute@ed3d-plugins-hailz
/plugin install ed3d-house-style@ed3d-plugins-hailz
# ... etc
```

## Repository Structure

```
ed3d-plugins-hailz/
├── .claude-plugin/
│   └── marketplace.json
├── plugins/
│   ├── ed3d-00-getting-started/
│   ├── ed3d-plan-and-execute/
│   ├── ed3d-house-style/
│   ├── ed3d-basic-agents/
│   ├── ed3d-research-agents/
│   ├── ed3d-extending-claude/
│   ├── ed3d-playwright/
│   ├── ed3d-hook-skill-reinforcement/
│   ├── ed3d-hook-claudemd-reminder/
│   ├── ed3d-hook-security-hardening/
│   └── ed3d-session-reflection/
└── README.md
```

## Contributing
Issues and pull requests gratefully solicited, except `ed3d-house-style` is _my_ house style, and provided for reference, so I might not take contributions there. (You can make your own house-style plugin though and use that instead!)

## Attribution

This repository is a fork of [`ed3dai/ed3d-plugins`](https://github.com/ed3dai/ed3d-plugins) by Ed. Most of the structure, naming, and core skills are Ed's work; this fork adds my personal modifications on top.

`ed3d-plan-and-execute` and parts of `ed3d-extending-claude` are originally derived from [`obra/superpowers`](https://github.com/obra/superpowers) by Jesse Vincent. The original plugin has been folded, spindled, and mutilated extensively.

Some skills in `ed3d-house-style` are derived from `obra/superpowers` and others (`property-based-testing` is a big one) are derived from the [Trail of Bits Skills repository](https://github.com/trailofbits/skills).

## License

The original [obra/superpowers](https://github.com/obra/superpowers) code in this repository is licensed under the MIT License, copyright Jesse Vincent. See `plugins/ed3d-plan-and-execute/LICENSE.superpowers`.

All other content is licensed under the [Creative Commons Attribution-ShareAlike 4.0 International License](http://creativecommons.org/licenses/by-sa/4.0/).