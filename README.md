# quarkus-rest

An opinionated [Claude Skill](https://agentskills.io/specification) for JAX-RS endpoints in Quarkus applications. When an AI coding agent is writing or modifying a `@Path` resource, this skill loads into context and tells it which `Response` to build, which status code to pick, where validation goes, and how to translate domain exceptions into HTTP responses without `try/catch` clutter in every method.

The full guidance lives in [`SKILL.md`](./SKILL.md). The short version:

- **Always** return `jakarta.ws.rs.core.Response`. Never return the entity / DTO directly — status codes get implicit and inconsistent.
- Resources are thin: validate input, delegate to an application service, translate the result. No repositories, no transactions, no domain logic.
- Use `@Valid` on bodies and `@NotNull` / `@NotBlank` / `@Size` / `@Email` on params and DTO fields. Bean Validation produces 400 automatically.
- Pick the status code consciously: 201 + `Location` for creates, 204 for no-body updates and deletes, 404 for missing, not 200 for everything.
- Map domain exceptions to HTTP via `@Provider ExceptionMapper<E>`, one per exception type. No `try/catch` for domain exceptions in resource methods.
- Endpoint paths are consistent across the service. Pick `/orders` *or* `/api/orders` and apply it everywhere — two paths is forever.
- Log only at validation failures and 5xx mapping. The application service logs state changes; the resource is the wrong layer for that.

The skill also ships an Excuse / Reality table for resisting the rationalizations that show up at 11 PM the night before a demo: "B is good enough, it returns Response," "expose the entity now, prettify later," "two paths won't matter, we'll standardize next sprint."

## Installation

The skill is a single `SKILL.md` file with YAML frontmatter. Drop the directory containing it into your agent's skills location.

### Claude Code

User-level (active across every project):

```bash
git clone https://github.com/<your-org>/quarkus-rest.git ~/.claude/skills/quarkus-rest
```

Project-level (active only in one project):

```bash
git clone https://github.com/<your-org>/quarkus-rest.git <your-project>/.claude/skills/quarkus-rest
```

Replace `<your-org>` with the GitHub owner once the repo is published.

### Other agents

| Agent | Skills directory |
|---|---|
| Codex | `~/.agents/skills/quarkus-rest/` |
| Copilot CLI | follow Copilot's plugin install docs; the skill is loaded by the same `Skill` tool |
| Gemini CLI | follow Gemini's `activate_skill` docs |

Only the directory containing `SKILL.md` is required. The other files in this repo (`README.md`, `LICENSE`) are repo metadata and don't affect skill loading.

## How triggering works

Agents read the `description:` field in `SKILL.md`'s YAML frontmatter to decide whether to load the body of the skill into context. The current trigger fires when the agent is:

- writing or editing a JAX-RS resource class (`@Path` annotated)
- choosing the right HTTP status code or constructing a `Response` object
- wiring up Bean Validation on request parameters or bodies
- adding an `ExceptionMapper` to translate a domain exception
- reviewing a diff that returns an entity directly, throws raw `WebApplicationException`, or builds responses inconsistently

If you want to broaden or narrow that trigger, edit the `description:` field — that's the part agents read every time.

## Companion skills

This skill pairs naturally with:

- **[quarkus-logging](https://github.com/<your-org>/quarkus-logging)** — for the log calls inside resource methods and exception mappers.
- **[quarkus-testing](https://github.com/<your-org>/quarkus-testing)** — for `@QuarkusTest` against the resource, REST Assured assertions on status and body.
- **[quarkus-persistence](https://github.com/<your-org>/quarkus-persistence)** — for the application service the resource delegates to.

## Authoring methodology

This skill was developed and verified using the [`superpowers:writing-skills`](https://github.com/anthropics/claude-code-plugins) RED-GREEN-REFACTOR workflow:

1. **RED** — run a multi-pressure scenario against a subagent without the skill loaded; capture the verbatim rationalizations the agent uses to justify the wrong choice.
2. **GREEN** — write or edit the skill addressing those specific rationalizations; re-run; verify the agent now picks the right option and cites the rule.
3. **REFACTOR** — capture any new rationalizations that surface and add them to the rule list, the Anti-patterns table, or the Excuse / Reality table.

The Excuse / Reality table in `SKILL.md` is built from rationalizations real subagents produced under simulated demo / executive / frontend-team pressure.

## License

MIT — see [`LICENSE`](./LICENSE).

## Contributions

PRs welcome, especially:

- Pressure-test scenarios that surface rationalizations the current skill doesn't address.
- Coverage for newer JAX-RS / Quarkus REST APIs (Server-Sent Events, async resources, multipart).
- HTTP-status guidance for adjacent domains (file uploads, batch operations, polling).

When proposing a rule change, please include the pressure scenario you used to validate it — the methodology section above describes the format.
