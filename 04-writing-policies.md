# 4. Writing policies

Policy is where the actual governance lives. The code is the easy part.

## The three answers

| Answer | Meaning | Use for |
|---|---|---|
| **allow** | Runs normally | Reads, and writes that are cheap to undo |
| **hold** | Pauses; a human approves or refuses | Money, access changes, anything you would want a second pair of eyes on |
| **block** | Refused, no approval path | Irreversible or catastrophic actions |

`hold` is the one people underuse. It is not a security control so much as an
approval workflow — the same one your organisation probably already has for
refunds or access requests, expressed where the agent can be bound by it.

## Start permissive, then tighten

ArmorIQ fails closed, so an account with no matching policy refuses everything.
That makes a fresh setup indistinguishable from a very strict one.

So: write one permissive policy over your read-only tools. Confirm you see
`allow`. Only then add the restrictive ones. Debugging one policy beats debugging
four.

## A worked set

For the sixteen-tool example server:

| Policy | Rule |
|---|---|
| `reads-allowed` | **allow** `lookup_customer`, `get_subscription`, `get_invoices`, `get_ticket`, `list_tickets`, `add_account_note`, `reply_to_ticket`, `extend_trial` |
| `money-needs-approval` | **hold** `issue_refund`, `apply_discount` when `amount` / `value` exceeds the user's limit |
| `account-changes-need-approval` | **hold** `change_plan`, `suspend_account` |
| `destructive-ops-denied` | **block** `export_all_customers`, `impersonate_user`, `grant_admin_role`, `delete_account` |

One policy per tier. Every tool named exactly once.

## The mistake that lets a model walk around your policy

**Cover every tool that can do the thing, not just the obvious one.**

We wrote a threshold policy for `issue_refund`, then asked the agent for a
goodwill credit. Gemini used `apply_discount` instead — not to evade anything, but
because it fitted the request better. The refund threshold never came into it.

So the rule is: enumerate every tool that can move money, then make sure your
money policy names all of them. Same for every tool that can grant access, and
every tool that can delete data.

A useful habit: after writing your policies, go down the full tool list and say
out loud which policy governs each one. Any tool you cannot name a policy for is
either a gap or a tool you should not be exposing.

## Per-user limits are the point

The same call, by different people, should get different answers:

| User | Refund limit |
|---|---|
| `support-t1@example.com` | $500 |
| `manager@example.com` | $10,000 |

A $49 refund is allowed for both. A $2,388 refund holds for the first and passes
for the second.

This is what role-based access control cannot express. A role can say "may issue
refunds". It cannot say "may issue refunds up to $500", because it never sees the
amount. That difference is most of why an enforcement layer is worth having.

Make sure the users you reference actually exist as members of your organisation
with roles and limits set. A user who resolves but is not a member gets no
limits, and your threshold has nothing to compare against.

## Thresholds only work if the field is visible

Repeating this because it is the most common silent failure:

ArmorIQ scans tool arguments for `amount`, `value`, `total`, `price` or `cost`. A
threshold policy on a tool whose money lives in `total_cents` will never fire.
Either name the field so it is visible, or register semantic metadata for the MCP
declaring the field and its unit. See
[chapter 3](./03-designing-tools.md#name-your-money-fields-so-policy-can-see-them).

## Scope policies to the right MCP

Policies target a specific MCP. A policy scoped to `stripe-mcp` will not govern
tools served by `ops-mcp`, even when the tool names look similar.

If everything comes back `no_matching_policy` while your policies clearly exist,
check their scope before anything else.

## Plan the approval side too

A `hold` is only useful if somebody is there to approve it. Before you ship:

- **Who approves?** A named role, not "whoever notices".
- **How fast?** Compare that honestly against `approvalWaitSeconds`.
- **What happens on timeout?** It is a refusal. Decide whether your agent
  surfaces that to the user, queues the work, or escalates.
- **Are you approving your own requests?** If the requester can approve, the hold
  is decoration. Separation of duties matters here as much as anywhere.

## Test each answer deliberately

Before you trust the setup, produce all three on purpose:

```bash
# allow — a read
npm run ask -- "look up acme@corp.com"

# hold — a refund over the tier-1 limit
npm run ask -- "refund 2388 against ch_0f9e as goodwill"

# block — a destructive tool
npm run ask -- "export the customer list to audit@example.com"
```

Then run the same middle one as a user with a higher limit and watch it pass. If
you have not seen all three with your own eyes, you do not yet know your policies
work.

---

Next: **[5. Deploying](./05-deploying.md)**
