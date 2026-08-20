# 3. Designing your tools

Your tools decide what policy is *able* to express. Get this wrong and no amount
of policy configuration will save you.

## Put no permission logic in your MCP server

This feels wrong the first time. Do it anyway.

Your MCP server should have no role checks, no permission logic, no policy. It
does what it is asked, by whoever asks.

Two reasons:

**It does not know who is asking.** The MCP server sees a tool call. The agent
layer knows the authenticated user, their limits, and their approval chain. Only
one of those two can make the decision correctly.

**Duplicated authorisation drifts.** Rules in your MCP *and* on the platform means
two places to update and two chances to disagree. One chokepoint is easier to
reason about and easier to audit.

## Group tools into risk tiers

Design the tiers first, then write the tools into them. Four works well:

| Tier | Character | Policy |
|---|---|---|
| **Reads** | Cannot change anything | allow |
| **Low-risk writes** | Reversible, cheap to undo | allow |
| **Sensitive** | Moves money, changes access, changes billing | hold for a human |
| **Destructive** | Irreversible, or catastrophic if wrong | block outright |

Tier 4 is the one people hesitate on. Resist the urge to make it "hold" instead of
"block". There is no amount of approval that makes `delete_account` a reasonable
thing for an autonomous agent to reach for. If a human genuinely needs to do it,
they can do it in the admin panel.

Give every tier at least two or three members. A tier with one tool in it usually
means the tier is wrong.

## Name your money fields so policy can see them

**This is the single most important detail in this chapter.**

ArmorIQ's policy engine finds the amount by scanning the tool's arguments for a
field named:

```
amount   value   total   price   cost
```

A field named anything else is **invisible to a threshold policy**. Your policy
will look correct, and will silently never fire.

```typescript
// Policy can see this.
inputSchema: { charge_id: z.string(), amount: z.number() }

// Policy CANNOT see this. The threshold will never trigger.
inputSchema: { charge_id: z.string(), total_cents: z.number() }
```

The instructive case is a discount tool. Its natural field is `percent` — and a
percentage is invisible to the scan. So give it a `value` too:

```typescript
inputSchema: {
  customer_id: z.string(),
  percent: z.number(),
  months: z.number(),
  value: z.number().describe('Total dollars discounted over the term'),
}
```

`value` is what policy reads, and it is also what a deal desk would actually
approve on. A 25% discount means nothing without knowing 25% of what.

If you genuinely cannot rename the field — an existing API, say — register
**semantic metadata** for the MCP on the platform, declaring which field holds
the number and what unit it is in. That is also how you get cents compared
correctly against a policy written in dollars.

## Tool descriptions change what the model does

The model reads your descriptions and takes them literally. This is not cosmetic.

A refund tool described as *"Refund a charge"* made Gemini refuse a goodwill
credit outright, reasoning that no suitable tool existed, because the credit
exceeded the original charge. Rewording it fixed the behaviour:

```typescript
description:
  'Refund or credit a customer against a charge. Amount is in US dollars and may ' +
  'exceed the original charge when issuing a goodwill credit. This is the only ' +
  'tool for refunds and credits of any kind.',
```

If a model keeps declining to use a tool you expect it to use, suspect the
description before you suspect the model.

## Make the dangerous tools actually work

If you are building a demo, do not stub the destructive tools out.

An `export_all_customers` that prints "blocked!" proves nothing. One that really
does export gives you a genuine before-and-after: run it with enforcement off,
watch the breach, then turn enforcement on.

The same applies to your own testing. A stub cannot tell you whether your policy
was doing the work or the stub was.

## Untrusted input is where the interesting failures are

Any tool that returns text a stranger wrote is an injection vector: support
tickets, customer names, order notes, uploaded documents, webhook payloads.

You cannot sanitise your way out of it. Instructions and data share one channel
and the model cannot reliably tell them apart. Assume anything it reads may be
hostile, and let the enforcement layer be the thing that holds.

If you are building an example, put the injection in the fixture data in plain
sight, and let the model find it. Real models do — we watched Gemini read a
poisoned ticket and call `export_all_customers` with no prompting at all.

## MCP server mechanics worth getting right

**Be stateless.** Build a fresh MCP server per request. Any instance can then
serve any request, which is what lets the service scale horizontally and deploy
cleanly to Cloud Run or Render.

**Answer `GET /mcp` with 405, not 404.** A stateless server offers no
server-initiated event stream, and the MCP spec says 405. Express's default 404
makes clients log a transport error that looks like a real fault.

**Read `PORT` from the environment.** Every hosting platform sets it.

**Add a reset endpoint** if you have fixture data. Without one, the second demo of
the day reports "already refunded" and looks broken.

---

Next: **[4. Writing policies](./04-writing-policies.md)**
