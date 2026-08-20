# 2. The integration

## The whole thing

```typescript
import { ArmorIQADK } from '@armoriq/sdk/dist/integrations/google_adk';

// Once per process.
const armoriq = new ArmorIQADK({
  apiKey: process.env.ARMORIQ_API_KEY!,
  agentName: 'ops-copilot',       // must match the platform
  defaultMcpName: 'ops-mcp',      // must match the platform
  approvalWaitSeconds: 60,
});

// Once per request, bound to the person the agent is acting for.
const scope = await armoriq.forUser(userEmail, { goal: question });

scope.install(agent);
try {
  for await (const event of runner.runAsync({ /* ... */ })) { /* ... */ }
} finally {
  scope.uninstall(agent);
  await scope.close();
}
```

## What `install()` actually does

It sets three properties on your ADK agent:

```
Gemini picks tools
   │
   ├─ afterModelCallback   build a plan from the chosen tools, get it signed
   ├─ beforeToolCallback   allow / hold / block, per tool, per user
   └─ afterToolCallback    report the outcome to the audit trail
```

**It overwrites whatever was there.** If you had your own `beforeToolCallback` on
the agent, it is inert for the lifetime of the scope. `uninstall()` restores the
originals.

If you want your own logging alongside, you have two clean options:

- the `onEvent` callback (below), or
- a Runner-level `BasePlugin`, which is a different layer and cannot collide

Do not put your own logic on those three agent properties while a scope is
installed.

> A trap worth knowing: Runner plugins receive `{ tool, toolArgs, toolContext }`,
> while agent callbacks receive `{ tool, args, context }`. Same concepts,
> different key names.

## `forUser` is the important part

```typescript
const scope = await armoriq.forUser('support-t1@example.com', { goal: question });
```

Policy is evaluated **for that person**. The same tool, with the same arguments,
can be allowed for a manager and held for a tier-1 agent. That is the thing plain
role-based access control cannot do: a static role cannot see that this particular
refund is for $2,388 rather than $49.

Create a scope per request, from the authenticated user. Never reuse one scope for
two people.

## The lifecycle

```typescript
scope.install(agent);
try {
  // run the agent
} finally {
  scope.uninstall(agent);
  await scope.close();
}
```

`close()` ends the plan and flushes the audit trail. **Always** in a `finally`,
so a failed run still records what happened. Skipping it leaves the trail
incomplete and, in a long-running process, leaks a flush timer.

## Streaming decisions to a UI

A held tool blocks while it waits for a human. Without feedback your app looks
frozen. `onEvent` tells you what is happening as it happens:

```typescript
const scope = await armoriq.forUser(userEmail, {
  goal: question,
  onEvent: (kind, payload) => {
    switch (kind) {
      case 'hold':     showWaiting(payload.tool, payload.reason); break;
      case 'approved': showApproved(payload.tool); break;
      case 'block':    showBlocked(payload.tool, payload.reason); break;
      case 'timeout':
      case 'rejected': showRefused(payload.tool); break;
      case 'error':    showFailClosed(payload.tool, payload.error); break;
    }
  },
});
```

## Reading outcomes from the run itself

Refusals also arrive as ordinary ADK function responses, carrying an
`armoriq_enforcement` field:

```typescript
for await (const event of runner.runAsync({ /* ... */ })) {
  for (const call of getFunctionCalls(event)) {
    console.log('calling', call.name);
  }
  for (const response of getFunctionResponses(event)) {
    const result = response.response as Record<string, unknown>;
    if (result?.armoriq_enforcement) {
      console.log('refused:', result.armoriq_enforcement);
    }
  }
}
```

This is why a refusal does not end the conversation. ArmorIQ hands the refusal
**to the model**, so the agent can say "I could not do that, here is what I did
instead" — rather than throwing and killing the turn.

Lean into it. Tell your agent in its instructions not to retry a refused tool and
not to look for a way around it:

```
Some actions are governed by company policy. A tool may come back refused or
held for approval. If that happens, do not retry it and do not look for a way
around it. Explain it plainly and carry on with what you can do.
```

## Holds, and the timeout

A `hold` pauses execution and waits for a human to approve on the platform.

```typescript
new ArmorIQADK({ /* ... */ approvalWaitSeconds: 60 });
```

The SDK's default is **300 seconds**. Consider whether you want a request thread
parked for five minutes. And note the failure mode: **a timeout is a refusal, not
an allow.** If nobody clicks, the tool does not run.

## Everything fails closed

| Situation | Result |
|---|---|
| No policy matches | blocked |
| Cannot reach ArmorIQ | blocked |
| Approval times out | blocked |
| Tool was never in the signed plan | blocked |

That last one is a second, independent line of defence: a tool that was not part
of the signed plan is refused by the SDK locally, with no network call at all.
Policy is the second check, not the first.

## Plan signing, and what to expect in the audit trail

`afterModelCallback` turns the model's chosen tools into a plan and has it signed,
producing an intent token and a plan hash.

Plans grow as an agent works. Rather than minting a new token every turn, ArmorIQ
records a **signed delta chained onto the original token**. So expect one token per
run with several deltas — not one token per turn. If you are reading the audit
trail and wondering why there is only one token, that is why.

---

Next: **[3. Designing your tools](./03-designing-tools.md)**
