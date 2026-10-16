# .NET Agent Skills

Reusable Agent Skills that provide AI coding agents with practical guidance for building, testing, reviewing, and maintaining .NET applications.

Agent Skills are self-contained instruction sets stored as `SKILL.md` files. They give compatible agents focused, reusable guidance for a specific task or domain without hard-coding project-specific assumptions.

## Available Skills

| Skill | Description |
| --- | --- |
| [`dotnet-backend`](skills/dotnet-backend/) | Production-oriented guidance for designing, implementing, refactoring, and reviewing .NET backend code across ASP.NET Core APIs, dependency injection, async I/O, EF Core, transactions, external integrations, security, logging, and maintainable architecture. |
| [`dotnet-testing`](skills/dotnet-testing/) | Practical guidance for creating and reviewing automated .NET tests, including unit, integration, API, database, concurrency, idempotency, and ASP.NET Core `WebApplicationFactory` scenarios. |
| [`ef-core`](skills/ef-core/) | Practical guidance for designing, implementing, optimizing, troubleshooting, and reviewing Entity Framework Core persistence code. |
| [`code-review`](skills/code-review/) | Review guidance for .NET code changes, pull requests, patches, and proposed implementations. |
| [`architecture-review`](skills/architecture-review/) | Architecture review guidance for .NET application boundaries, dependencies, coupling, data flow, reliability, scalability, deployment, persistence, and integrations. |

## Pro Editions

The skills in this repository are free and will stay free under the MIT License.

Pro editions are available on [Agensi](https://www.agensi.io) for teams that want deeper guidance. Each Pro edition builds on the free skill and adds detailed reference material, production-readiness checklists, reusable templates, and helper scripts. Updates reach your agent automatically through the Agensi connector.

| Free skill | Pro edition |
| --- | --- |
| [`dotnet-backend`](skills/dotnet-backend/) | `dotnet-backend-pro` (coming soon) |
| [`dotnet-testing`](skills/dotnet-testing/) | `dotnet-testing-pro` (coming soon) |
| [`ef-core`](skills/ef-core/) | `ef-core-pro` (coming soon) |
| [`code-review`](skills/code-review/) | `code-review-pro` (coming soon) |
| [`architecture-review`](skills/architecture-review/) | `architecture-review-pro` (coming soon) |

Pro editions use distinct skill names, so you can install them next to the free skills. When both are available, the Pro edition takes precedence.

## Installation / Usage

The exact installation flow depends on the agent you use, but the general approach is the same:

1. Clone or download this repository.
2. Copy or reference the skill directory you want.
3. Place or configure it in the location your agent expects for Agent Skills.

```bash
git clone https://github.com/solakpetri/Enzo.Ai.Skills.git
```

Each skill is self-contained. You can use one skill without installing the others.

### OpenCode

OpenCode discovers skills from directories that contain `SKILL.md` files.

The simplest approach is to copy one or more skill directories into a supported OpenCode skills location, preserving the directory structure. Common locations are project-local `.opencode/skills/` and global `~/.config/opencode/skills/`:

```text
.opencode/skills/
├── dotnet-backend/
│   └── SKILL.md
├── dotnet-testing/
│   └── SKILL.md
├── ef-core/
│   └── SKILL.md
├── code-review/
│   └── SKILL.md
└── architecture-review/
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

Skills are not commands. Once a skill is available to an agent, developers can usually prompt normally and let the agent use the task context to decide what guidance applies. For example, a request like this may be enough for an agent that automatically discovers applicable skills:

```text
Add an endpoint for creating orders.
```

Loading and discovery behavior depends on the agent being used, so follow your agent's documentation for details.

## Usage Examples

Once a skill is available to the agent, developers can prompt normally. The skill provides specialized context and guidance for the task without requiring developers to duplicate the full skill instructions in every prompt.

### .NET Backend

[`dotnet-backend`](skills/dotnet-backend/)

```text
Add a POST /orders endpoint that creates a new order.

Inspect the existing solution first and follow its architecture and conventions.

Validate the request, persist the order, handle expected errors, use async APIs and CancellationToken, and add relevant tests.

Build the affected projects and run the relevant tests when finished.
```

### .NET Testing

[`dotnet-testing`](skills/dotnet-testing/)

```text
Add tests for order submission.

An order with no order lines must not be submitted.

Inspect the existing test projects and follow their framework and conventions.

Cover the successful case and important failure cases. Choose the smallest test level that reliably proves the behavior and avoid unnecessary mocking.

Run the relevant tests when finished.
```

### EF Core

[`ef-core`](skills/ef-core/)

```text
Add a query that returns the 20 most recent orders for a customer.

Return only:
- order id
- created date
- status
- total amount

Inspect the existing persistence code first. Use an efficient projection, read-only query behavior where appropriate, deterministic ordering, and CancellationToken.

Avoid loading unnecessary entities or navigation properties.

Add relevant tests and run them.
```

### Code Review

[`code-review`](skills/code-review/)

```text
Review the changes in the current branch against the default branch.

Do not modify any files.

Focus on defects introduced by the change, especially correctness, data integrity, concurrency, security, API behavior, EF Core usage, and missing meaningful tests.

For each finding, provide the severity, location, concrete failure scenario, consequence, and recommended direction.

Do not manufacture findings if the change is sound.
```

### Architecture Review

[`architecture-review`](skills/architecture-review/)

```text
Review the architecture of this .NET solution.

Do not modify any files.

Evaluate module and service boundaries, dependency direction, data ownership, persistence boundaries, integrations, reliability, scalability, and operational complexity.

Identify concrete architectural risks and explain their tradeoffs.

Do not recommend microservices or additional abstractions unless they solve a demonstrated problem.
```

## Repository Structure

```text
dotnet-agent-skills/
├── skills/
│   ├── architecture-review/
│   │   └── SKILL.md
│   ├── code-review/
│   │   └── SKILL.md
│   ├── dotnet-backend/
│   │   └── SKILL.md
│   ├── dotnet-testing/
│   │   └── SKILL.md
│   └── ef-core/
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

This project is licensed under the [MIT License](LICENSE).
