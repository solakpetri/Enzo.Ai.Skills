# .NET Agent Skills

Reusable Agent Skills that provide AI coding agents with practical guidance for building, testing, reviewing, and maintaining .NET applications.

Agent Skills are self-contained instruction sets stored as `SKILL.md` files. They give compatible agents focused, reusable guidance for a specific task or domain without hard-coding project-specific assumptions.

## Available Skills

| Skill | Description |
| --- | --- |
| [`dotnet-backend`](skills/dotnet-backend/) | Production-oriented guidance for designing, implementing, refactoring, and reviewing .NET backend code across ASP.NET Core APIs, dependency injection, async I/O, EF Core, transactions, external integrations, security, logging, and maintainable architecture. |
| [`dotnet-testing`](skills/dotnet-testing/) | Practical guidance for creating and reviewing automated .NET tests, including unit, integration, API, database, concurrency, idempotency, and ASP.NET Core `WebApplicationFactory` scenarios. |

## Installation / Usage

The exact installation flow depends on the agent you use, but the general approach is the same:

1. Clone or download this repository.
2. Copy or reference the skill directory you want.
3. Place or configure it in the location your agent expects for Agent Skills.

```bash
git clone https://github.com/solakpetri/Enzo.Ai.Skills.git
```

Each skill is self-contained. You can use one skill without installing the other.

### OpenCode

OpenCode discovers skills from directories that contain `SKILL.md` files.

The simplest approach is to copy one or both skill directories into a supported OpenCode skills location, preserving the directory structure. Common locations are project-local `.opencode/skills/` and global `~/.config/opencode/skills/`:

```text
.opencode/skills/
├── dotnet-backend/
│   └── SKILL.md
└── dotnet-testing/
    └── SKILL.md
```

OpenCode also supports registering an external skills directory through `skills.paths` in `opencode.json`:

```json
{
  "$schema": "https://opencode.ai/config.json",
  "skills": {
    "paths": ["/path/to/dotnet-agent-skills/skills"]
  }
}
```

After adding or changing skills, restart OpenCode so it reloads configuration and skills.

### Other Agents

These skills use the Agent Skills `SKILL.md` structure. Other agents that support the same format, or a compatible one, may be able to use them with little or no modification.

## Repository Structure

```text
dotnet-agent-skills/
├── skills/
│   ├── dotnet-backend/
│   │   └── SKILL.md
│   └── dotnet-testing/
│       └── SKILL.md
└── README.md
```

Each skill lives in its own directory and is meant to be reusable on its own.

## Design Principles

- Practical guidance over framework documentation.
- Production-oriented recommendations.
- Respect existing repository conventions before introducing new patterns.
- Avoid unnecessary abstractions.
- Focus on decisions AI coding agents actually need to make.
- Stay framework- and library-neutral where that improves reuse.

## Contributing

Contributions are welcome for:

- improving existing skills
- correcting inaccurate guidance
- adding useful examples
- proposing new .NET-focused skills

Please keep skills:

- focused
- actionable
- technically accurate
- broadly applicable
- concise enough to be useful as agent context

## License

Free :)
