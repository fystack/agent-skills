# Fystack agent skills

Agent skills that teach AI coding assistants (Claude Code, Cursor, Codex, Gemini CLI, GitHub Copilot and others that support the [Agent Skills](https://agentskills.io) format) how to build on [Fystack](https://fystack.io).

## Skills

**`fystack`**: build on Fystack from any language. It covers:

- the REST API: an endpoint map, plus request signing with HMAC or Ed25519
- the official TypeScript (`@fystack/sdk`) and Go (`github.com/fystack/platform-sdk-go`) SDKs
- webhook verification
- the product model: workspaces, roles, approval groups, policies, Gas Station, sweeps and alerts. This lets it answer setup questions as well as write code.

It also builds in guardrails for custody code: sandbox first, no secrets in code, and no bypassing human approval of funds.

The skill stays small and points the agent at the live docs (`https://docs.fystack.io/llms.txt` and the Markdown version of every page), so it does not go stale when the API changes.

## Install

Any agent, with the [skills CLI](https://github.com/vercel-labs/skills):

```bash
npx skills add fystack/agent-skills
```

Claude Code, as a plugin:

```
/plugin marketplace add fystack/agent-skills
/plugin install fystack@fystack
```

Manually: copy `skills/fystack/` into your agent's skills folder, for example `.claude/skills/fystack/` in your project or `~/.claude/skills/fystack/` for all projects.

## Try it

After installing, ask your assistant something like:

- "Create an MPC wallet in the Fystack sandbox and print its EVM deposit address."
- "Add an Express endpoint that receives Fystack webhooks, verifies the signature and credits confirmed deposits once."
- "Write a Python client for the Fystack API that signs requests with our Ed25519 key."
- "How do we set it up so withdrawals over $10k need 2 of 3 finance approvers?"

## Layout

```
skills/fystack/
├── SKILL.md                    # workflow, core facts, guardrails, doc map
└── references/
    ├── api.md                  # every REST endpoint with key fields and statuses
    ├── authentication.md       # request signing in TypeScript, Python and Go
    ├── concepts.md             # workspaces, roles, approval groups, policies, ops features
    ├── webhooks.md             # signature verification and safe event handling
    ├── sdk.md                  # @fystack/sdk (TypeScript) surface and known doc drift
    └── sdk-go.md               # platform-sdk-go (Go) surface
.claude-plugin/marketplace.json # Claude Code plugin marketplace entry
```

`evals/` holds the test prompts and pass/fail checks used to measure the skill against an agent without it.

## Contributing

Keep `SKILL.md` short and link to the docs for details. When the API or an SDK changes, update the facts here and in the [docs](https://docs.fystack.io) together.

## License

[MIT](LICENSE)
