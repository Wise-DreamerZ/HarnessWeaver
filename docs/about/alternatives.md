# Alternatives and comparisons

HarnessWeaver puts controls around the model calls, tool calls and outputs of an app that already has an executor, from one spec merged with an org baseline whose locks no app that extends it can remove. This page has two parts: the [alternatives](#alternatives), which a team might choose instead of HarnessWeaver, and the tools it [works with](#works-with), which do a job it leaves to them.

## Alternatives

Each of these covers part of what HarnessWeaver does.

| Product | What it does | How HarnessWeaver differs |
| --- | --- | --- |
| NeMo Guardrails | Programmable rails for an LLM app: input, dialog, retrieval, execution and output, configured in YAML and Colang | No dialog flows. One spec also holds budgets, typed output, tool governance, approvals and evals, under an org baseline whose locks apps can't remove |
| Guardrails AI | Validators from the Guardrails Hub, combined into input and output guards | A detector is one provider. The spec decides what happens when it fires, where, and whether it enforces |
| LangChain middleware, OpenAI Agents SDK guardrails | Controls built into one agent framework, written in the app's code | One spec for every executor and language, and a platform baseline that applies whatever the app's code does |
| Microsoft Agent Governance Toolkit | Runtime governance for fleets of agents: deterministic policy on tool calls and messages between agents, agent identity, isolation rings, a kill switch, tamper-evident audit | Governs each run's model calls, tools and output, with budgets, approvals, typed output and evals. Agent identity, sandbox rings and compliance evidence are out of its scope |
| Aegis | A gateway every tool call and model call is routed through: detectors, anomaly scores, an approval queue and a hash-chained audit log | A framework each team composes into its own harness, not one service to route through. It also owns budgets, typed output, guards on output and evals |

### A framework, not a central harness

Many teams build one central harness and route every app through it. HarnessWeaver is a framework each team uses to build *its own* harness, while the platform team's baseline still applies to all of them.

| | A central harness | HarnessWeaver |
| --- | --- | --- |
| Who owns control flow | The harness | Your app, with HarnessWeaver intercepting |
| How controls are applied | Hard-wired in one runtime | Composed per app from plugins, plus an org baseline that can't be removed |
| Adopting it | Route your app through it | One line for tracing, then grow level by level |
| Languages | Usually one | Python, TypeScript, and any language through the sidecar |

A central gateway has one strength HarnessWeaver doesn't: every call must pass through it, so its controls apply even to code that never asked for them. HarnessWeaver runs inside the app, so a client it never instrumented or wrapped is not governed.

### Guardrail frameworks

These are the closest alternatives.

**NeMo Guardrails** adds programmable rails to an LLM app: input rails on the user's message, dialog rails that steer the conversation with Colang flows, retrieval rails on retrieved chunks, execution rails on tool calls, and output rails on the answer. HarnessWeaver has no dialog rails or flow language, because it never decides what the conversation says next (RULES R12). Its guards run at the same places (input, retrieved context, tool results, output and streams), and the same spec also holds budgets and deadlines, typed output with repair, tool manifests and the governor, policy, approvals and evals. A platform team's baseline is merged into every spec that extends it, and no such app can remove its locked controls.

**Guardrails AI** combines validators from the Guardrails Hub into input and output guards around a model call, and can run them as a server that apps call. In HarnessWeaver a detector is a `Guardrail` **provider**: it reports what it found, and the spec decides what happens, at which stage, in which band, in shadow mode or enforced, and with what fail mode. The built-in guards are deterministic and need no dependencies. Model-based detectors (Presidio, Llama Guard, Prompt Guard) are *planned* as plugins, and any validator library can be wrapped as a provider today ([Providers and ports](../advanced/providers.md#example-a-guard)).

### Controls built into agent frameworks

Some agent frameworks have controls of their own. LangChain agents take middleware that redacts personal data before the model sees it, asks a person to approve a tool call, or runs before each model call and around each tool call. The OpenAI Agents SDK has input, output and tool guardrails whose tripwires stop the run. These controls live in the app's code and work inside that one framework. HarnessWeaver's controls live in a spec that applies the same way to raw SDK calls, to the frameworks it has adapters for, and to other languages, and the org baseline applies whether or not the app's code asks for it.

### Agent governance and tool firewalls

These sit in the path of an agent's actions, mostly its tool calls, and decide whether each one runs.

**[Microsoft Agent Governance Toolkit](https://github.com/microsoft/agent-governance-toolkit)** puts deterministic policy, written in YAML, OPA/Rego or Cedar, in front of every tool call, resource access and message between agents, through adapters for many agent frameworks and SDKs in Python, TypeScript, .NET, Rust and Go. Around that it adds a cryptographic identity and trust score for each agent, isolation rings, compensation for multi-step work, a kill switch, reliability budgets and circuit breakers, and tamper-evident audit logs mapped to compliance frameworks. It governs the fleet of agents; HarnessWeaver governs each app's runs, and has no agent identity, sandbox rings or compliance evidence.

**[Aegis](https://github.com/Justin0504/Aegis)**, an open-source firewall for agent tool calls, is a gateway with a dashboard. SDKs that patch model and framework clients, HTTP proxies for model APIs and an MCP proxy send each call to it. It classifies the call, scores it against the agent's learned behavior, runs its detectors and per-tenant rules, and allows the call, blocks it or holds it for a person in its approval queue. Every decision goes into a hash-chained audit log, and, like HarnessWeaver's tool manifests, its MCP proxy pins each tool's definition and refuses one that changed. Aegis is a [central harness](#a-framework-not-a-central-harness). HarnessWeaver also owns what a tool firewall doesn't: per-run budgets and deadlines, typed output with repair, guards on output and streams, shadow mode for new guards (RULES R8) and evals. As in HarnessWeaver (RULES R3), anomaly scores and LLM checks can only tighten its decisions.

## Works with

These are not alternatives. Each does a job HarnessWeaver leaves to it, and the two are used together.

| Tool | Its job | How HarnessWeaver works with it |
| --- | --- | --- |
| Agent frameworks: Pydantic AI, the Vercel AI SDK | The app's reasoning: the prompts, the agent loop, which tool to call next. HarnessWeaver does none of that (RULES R12) | Governs them through [adapters](../advanced/runners-and-adapters.md#adapters): Pydantic AI under the `pydantic_ai.agent` runner or `harness.wrap(agent)`, Vercel AI SDK models with `wrapModel`. Adapters for LangGraph and the OpenAI Agents SDK are *planned* |
| Model gateways: [LiteLLM Proxy](https://docs.litellm.ai/docs/simple_proxy) | Provider keys, routing, retries, fallbacks, tenant budgets | `openai.chat` talks to it and leaves retries to it, so each control has one owner (RULES R5). `harnessweaver gateway litellm` writes its config ([Gateways](../deployment/gateways.md)) |
| MCP gateways, such as [Preloop](https://preloop.ai/) | Allow, deny or hold each MCP tool call for a person's approval | The tool governor makes that decision inside the app. For apps that reach tools through the gateway alone, `harnessweaver gateway mcp` writes the spec's tool allow-list for it to enforce ([Gateways](../deployment/gateways.md#gateway-mcp)) |
| Observability backends: Langfuse, Phoenix, Logfire | Store and show traces | Produces OpenTelemetry spans and metrics in the GenAI semantic conventions, with every guard, rule, budget check and approval as a traced decision and no content unless captured after redaction. `harnessweaver dev` wires a local collector to Langfuse ([Observability](../tutorial/observability.md)) |
