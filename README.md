# Building governed agents with Google ADK and the ArmorIQ SDK

A practical guide. Everything here was learned by building the two example repos
that go with it, so the warnings are things that actually went wrong rather than
things that theoretically might.

| Repo | |
|---|---|
| [`armoriq-adk-ops-mcp`](https://github.com/armoriq/armoriq-adk-ops-mcp) | a working MCP server — sixteen tools in four risk tiers |
| [`armoriq-adk-ops-agent`](https://github.com/armoriq/armoriq-adk-ops-agent) | a working ADK agent, governed by ArmorIQ |
| **this repo** | how to build your own |

---

## The idea in one paragraph

An agent decides what to do; something else decides whether it is allowed to.
ArmorIQ sits at the tool boundary — after the model has chosen, before the tool
runs — and answers **allow**, **hold for a human**, or **block**, based on policy
and on *which person* the agent is acting for. Because that decision happens
outside the model, it does not matter what the model was persuaded to do.

That last point is the whole value. Prompt injection, jailbreaks, a confused
model, a malicious user — none of it changes the answer, because the answer was
never the model's to give.

## The integration, in full

```typescript
import { ArmorIQADK } from '@armoriq/sdk/dist/integrations/google_adk';

const armoriq = new ArmorIQADK({ apiKey, agentName, defaultMcpName });
const scope = await armoriq.forUser(userEmail, { goal: question });

scope.install(agent);
try {
  for await (const event of runner.runAsync({ /* ... */ })) { /* ... */ }
} finally {
  scope.uninstall(agent);
  await scope.close();
}
```

That is not a simplified snippet. That is the integration. Everything else in
your agent stays ordinary ADK code.

`install()` attaches three callbacks to the ADK agent:

| Callback | What ArmorIQ does |
|---|---|
| `afterModelCallback` | The model has chosen its tools. Describe them as a plan, and get that plan cryptographically signed. |
| `beforeToolCallback` | Before each tool runs: allow, hold, or block. |
| `afterToolCallback` | Record what happened, for the audit trail. |

## The chapters

| | |
|---|---|
| **[1. Getting started](./01-getting-started.md)** | Accounts, keys, registering your MCP and agent, first policies |
| **[2. The integration](./02-the-integration.md)** | The three callbacks in detail, the per-user scope, the lifecycle, streaming decisions to a UI |
| **[3. Designing your tools](./03-designing-tools.md)** | Risk tiers, naming money fields so policy can see them, why your MCP should have no permission logic |
| **[4. Writing policies](./04-writing-policies.md)** | Allow, hold, block. Thresholds, per-user limits, and the mistake that lets a model route around your policy |
| **[5. Deploying](./05-deploying.md)** | Two services, separately deployed. Render, Cloud Run, and the packaging mistake that stops a container booting |
| **[6. Troubleshooting](./06-troubleshooting.md)** | Every wrong turn we hit, and what it looked like |
| **[7. Production checklist](./07-production-checklist.md)** | What to fix before this faces real users |

## What you need

| | |
|---|---|
| Node | 20 or newer |
| An ArmorIQ account | [platform.armoriq.ai](https://platform.armoriq.ai) — for the API key, and to register your MCP, agent and policies |
| A model | Google ADK is Gemini-native. A key from [aistudio.google.com/apikey](https://aistudio.google.com/apikey), or Vertex AI |
| Somewhere to deploy | Your MCP server needs a public HTTPS URL, because the ArmorIQ platform calls it |

## The shape of a project

Two services, deployed separately, registered separately:

```
your-mcp-server/     the tools     → register as an MCP
your-agent/          ADK + ArmorIQ → register as an agent
```

Keep them apart. The MCP server holds no security logic at all — governance lives
at the agent layer, which is the only place that knows *who* is asking.

## Three things to know before you start

**Everything fails closed.** No matching policy means blocked. A network failure
reaching ArmorIQ means blocked. An approval that times out means blocked. This is
correct behaviour, but it does mean a half-configured setup looks identical to a
very strict one. See [chapter 6](./06-troubleshooting.md).

**A refused tool comes back *into* the model.** ArmorIQ returns the refusal as the
tool's result rather than throwing, so your agent can read it, explain itself, and
carry on with what it is still allowed to do. Refusal does not have to mean a dead
conversation.

**The model is not deterministic, and that is the point.** Across repeated runs of
the same prompt, Gemini does not always choose the same tools. The enforcement
layer does not care — which is exactly why enforcement does not live in your
prompt.
