# 6. Troubleshooting

Every one of these actually happened while building the example repos.

---

## Every tool comes back `no_matching_policy`

**Working as designed.** ArmorIQ fails closed, so no matching policy means
blocked.

Check, in order:

1. Does a policy exist for these tools?
2. Is it scoped to the **right MCP**? A policy on `stripe-mcp` will not govern
   `ops-mcp`.
3. Does `ARMORIQ_MCP_NAME` match the registered MCP name exactly?
4. Does `ARMORIQ_AGENT_NAME` match a registered agent exactly?

Items 3 and 4 are the usual culprits, and neither produces an error.

---

## My policies exist but do not seem to apply

Almost always the agent name. Policy is attributed by agent name, so if your code
sends `ops-copilot` and the platform has `my-agent`, nothing matches and nothing
complains.

Confirm with `bootstrap()`, which lists the registered agents. It is a read, so
it is safe against a live account.

---

## A threshold policy never fires

The amount field is invisible. ArmorIQ scans arguments for `amount`, `value`,
`total`, `price` or `cost`. A field called `total_cents`, `sum` or `percent` is
not found, so there is nothing to compare and the threshold is skipped.

Rename the field, or register semantic metadata for the MCP declaring which field
holds the number and its unit.

---

## The model used a different tool and skipped my policy

Not evasion — the model picked whichever tool fitted the request.

We put a threshold on `issue_refund`, asked for a goodwill credit, and Gemini
used `apply_discount` instead. The threshold was never involved.

Enumerate every tool that can do the thing, and name all of them in the policy.

---

## `Cannot find module '@armoriq/sdk/integrations/google_adk'`

Use the `/dist/` path:

```typescript
import { ArmorIQADK } from '@armoriq/sdk/dist/integrations/google_adk';   // works
import { ArmorIQADK } from '@armoriq/sdk/integrations/google_adk';        // does not
```

The SDK is CommonJS and publishes no `exports` map, so only the real on-disk path
resolves. Some SDK docstrings show the shorter form; it does not work.

Also set `"moduleResolution": "bundler"` in `tsconfig.json`.

---

## The container builds but will not start

`tsx` is in `devDependencies`. Production installs use `--omit=dev`, so the image
has no `tsx` and the start command fails.

Move `tsx` to `dependencies`. See [chapter 5](./05-deploying.md).

---

## My own `beforeToolCallback` stopped running

`scope.install(agent)` overwrites all three agent callbacks. Yours is inert until
`uninstall()`.

Use the `onEvent` option, or a Runner-level `BasePlugin` — a different layer that
cannot collide.

---

## The agent runs, calls nothing, and returns nothing

Usually a bad or missing model credential. The model call fails, ADK logs it as an
error, and the run completes having done nothing.

Two things make this much less painful:

- Do not silence ADK's logging entirely. It logs through winston, straight to
  stdout, so patching `console` will not reach it — use its exported
  `setLogger()`, and keep errors.
- Detect a run that made zero tool calls and say so explicitly, with the likely
  cause. An empty result with no explanation is the worst possible output.

And check the variable name: `@google/genai` reads `GOOGLE_GENAI_API_KEY` or
`GEMINI_API_KEY`, **not** `GOOGLE_API_KEY`.

---

## A held tool hangs for five minutes

That is `approvalWaitSeconds`, defaulting to 300. Lower it:

```typescript
new ArmorIQADK({ /* ... */ approvalWaitSeconds: 60 });
```

Remember a timeout is a **refusal**, not an allow.

---

## `MCP transport error: Failed to open SSE stream: Not Found`

Your stateless MCP server is answering `GET /mcp` with 404. The spec says 405.
Add a handler:

```typescript
app.get('/mcp', (_req, res) => {
  res.status(405).set('Allow', 'POST').json({
    jsonrpc: '2.0',
    id: null,
    error: { code: -32000, message: 'This server is stateless. Use POST.' },
  });
});
```

Harmless, but it fills your logs with a fault that is not one.

---

## The second demo run says "already refunded"

Your MCP server is holding state from the first run. Add a `POST /reset` that
restores the fixture data, and call it at the start of each run.

---

## The first request after a quiet period takes 30 seconds

A free-tier host has spun the service down. From the agent's side this looks like
a hang. Use a paid instance for anything you demonstrate live.

---

## The audit trail shows one token but the agent ran ten tools

Correct. Plan growth is recorded as a **signed delta chained onto the original
token**, not as a fresh token per turn. One token, several deltas, one run.

---

## The same prompt behaves differently each time

Expected. Across repeated runs of one prompt, Gemini does not always choose the
same tools — in our testing it hit all three enforcement outcomes on two runs out
of three, and on the third it did a couple of reads and stopped.

Two consequences:

- Do not build a demo that depends on the model choosing a specific tool. If you
  need determinism, supply a scripted model implementation — `LlmAgent.model`
  accepts a `BaseLlm` instance, so the rest of the pipeline stays real.
- Do not build *enforcement* that depends on the model behaving. That is the
  entire reason enforcement lives outside it.

---

## Tools the model refuses to use

Suspect the tool description before the model. A refund tool described as
"Refund a charge" made Gemini decline a goodwill credit larger than the original
charge, on the grounds that no suitable tool existed. Spelling out that the amount
may exceed the charge fixed it.

---

Next: **[7. Production checklist](./07-production-checklist.md)**
