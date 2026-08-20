# 1. Getting started

Roughly thirty minutes, most of it in the dashboard rather than in code.

## Install

```bash
npm install @google/adk @armoriq/sdk
```

Two things to get right in `tsconfig.json`:

```json
{
  "compilerOptions": {
    "module": "ESNext",
    "moduleResolution": "bundler"
  }
}
```

`@google/adk` is pure ESM. `@armoriq/sdk` is CommonJS and publishes no `exports`
map, so the ADK integration can only be reached by its real on-disk path:

```typescript
import { ArmorIQADK } from '@armoriq/sdk/dist/integrations/google_adk';   // works
import { ArmorIQADK } from '@armoriq/sdk/integrations/google_adk';        // does not
```

`"moduleResolution": "bundler"` is what makes that deep path resolve.

## Get your keys

| Key | Where | Notes |
|---|---|---|
| `ARMORIQ_API_KEY` | platform.armoriq.ai | Starts with `ak_live_` or `ak_test_`. The SDK validates the prefix when you construct the client, so a malformed key fails immediately rather than mysteriously. |
| `GEMINI_API_KEY` | aistudio.google.com/apikey | `@google/genai` reads `GOOGLE_GENAI_API_KEY` or `GEMINI_API_KEY` — **not** `GOOGLE_API_KEY`, which is the one most people reach for first. |

## Build the MCP server first

Your agent's tools come from an MCP server, and the ArmorIQ platform needs that
server's public URL. So build and deploy it before you wire up policy.

See [chapter 3](./03-designing-tools.md) for how to design the tools, and
[`armoriq-adk-ops-mcp`](https://github.com/armoriq/armoriq-adk-ops-mcp) for a working one.

It has to be reachable at a **public HTTPS URL**. The platform calls it directly,
so `localhost` will not work once you are running against the real thing.

## Register on the platform

Three things to register, in this order.

### 1. The MCP

MCP Registry → Add MCP:

| Field | Example |
|---|---|
| Name | `ops-mcp` |
| URL | `https://ops-mcp.onrender.com/mcp` |

The name is the identifier your code uses. It must match exactly.

### 2. The agent

Register an agent — say `ops-copilot`. Policy is attributed by agent name, so if
this does not match what your code sends, your policies will appear not to fire at
all. This is the single most common setup mistake.

### 3. Policies

Nothing works until a policy matches. **ArmorIQ fails closed**, so an account with
no policies for your MCP refuses every call with `no_matching_policy`.

Start with one permissive policy over your read-only tools, confirm you see
`allow`, then add the restrictive ones. Debugging one policy is much easier than
debugging four.

See [chapter 4](./04-writing-policies.md).

## Wire it up

```bash
ARMORIQ_API_KEY=ak_live_xxxxx
ARMORIQ_AGENT_NAME=ops-copilot          # must match the platform
ARMORIQ_MCP_NAME=ops-mcp                # must match the platform
MCP_URL=https://ops-mcp.onrender.com/mcp
GEMINI_API_KEY=xxxxx
```

## Verify before you debug

Write a setup check and run it before you run the agent. It saves hours, because
the two failure modes below look identical from inside a failed agent run.

```typescript
import { ArmorIQClient } from '@armoriq/sdk';

const client = new ArmorIQClient({
  apiKey: process.env.ARMORIQ_API_KEY!,
  userId: 'agent',
  agentId: process.env.ARMORIQ_AGENT_NAME!,
  useProduction: true,
});

const account = await client.bootstrap();

console.log('org: ', account.org?.name);
console.log('agents:', (account.agents ?? []).map((a: any) => a.name));
console.log('mcps:  ', (account.mcps ?? []).map((m: any) => m.name));
```

`bootstrap()` is a read — it mints nothing and creates no records, so it is safe
against a live account. Compare what it prints against your environment
variables. If either name is missing from those lists, stop and fix that first.

[`armoriq-adk-ops-agent`](https://github.com/armoriq/armoriq-adk-ops-agent) ships this as
`npm run check`, and it reports the mismatch in plain English.

## First run

Expect to be blocked. If every tool returns `no_matching_policy`, your policies
are missing or scoped to a different MCP — that is the designed behaviour, not a
bug. Go back and add the permissive read policy.

Once reads come back `allow`, you know the whole chain works: key, agent name,
MCP name, policy scope. Everything after that is just more policy.

---

Next: **[2. The integration](./02-the-integration.md)**
