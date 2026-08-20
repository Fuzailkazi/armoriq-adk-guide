# 7. Production checklist

Work through this before real users touch it.

## The MCP server

- [ ] **Authentication.** Deployed publicly with no auth, anyone who finds the URL
      can call your tools directly and bypass ArmorIQ entirely — enforcement is at
      the agent layer, not here. Add a bearer token or API key and register it as
      the MCP's auth on the platform.
- [ ] **No permission logic in the tools.** Authorisation belongs at the one
      chokepoint that knows who is asking. Two places to update means two places
      to drift.
- [ ] **Every money field is named `amount`, `value`, `total`, `price` or `cost`**
      — or has semantic metadata registered. Otherwise thresholds silently do not
      fire.
- [ ] **Destructive tools genuinely belong in the tool list.** If a human should
      always do it in the admin panel, do not expose it to the agent at all.
      Blocking a tool is good; not shipping it is better.
- [ ] **Rate limiting**, if it fronts anything expensive.
- [ ] **A production install boots.** `npm install --omit=dev && npm start` in a
      clean directory.

## The agent

- [ ] **`uninstall()` and `close()` in a `finally`.** `close()` flushes the audit
      trail; skipping it loses records and leaks a timer.
- [ ] **A scope per request, from the authenticated user.** Never reuse one scope
      across users, and never take the user's identity from a request body a
      client controls.
- [ ] **`approvalWaitSeconds` matches how fast your approvers actually respond.**
      The default is 300 seconds of a parked request thread.
- [ ] **Timeouts are handled.** A timeout is a refusal. Decide whether you surface
      it, queue the work, or escalate.
- [ ] **Instructions tell the model not to retry or work around refusals.**
- [ ] **A run that calls no tools is reported as such**, with a likely cause. Silent
      empty results are the hardest failure to diagnose.

## Policy

- [ ] **Every tool is governed by a named policy.** Walk the full tool list and say
      which policy covers each. Any tool you cannot name one for is a gap.
- [ ] **Every tool that can move money is in the money policy** — not just the
      obvious one. Models pick whichever tool fits.
- [ ] **Same for access grants and data deletion.**
- [ ] **Policies are scoped to the right MCP.**
- [ ] **Users exist as members with roles and limits.** A user who resolves but is
      not a member has no limits to compare against.
- [ ] **Approvers cannot approve their own requests.**
- [ ] **All three outcomes verified by hand** — allow, hold, block — plus the same
      held call passing for a higher-limit user.

## Secrets

- [ ] **Keys are platform secrets**, not committed environment files.
- [ ] **`.env` is in `.gitignore`.**
- [ ] **Any key that has ever appeared in a repo, a log, a screenshot or a chat
      transcript is rotated.** Deleting the commit is not enough — it is already
      in clones and caches.
- [ ] **Separate keys per environment**, so revoking staging does not take
      production down.

## Names and configuration

- [ ] **`ARMORIQ_AGENT_NAME` matches a registered agent, exactly.**
- [ ] **`ARMORIQ_MCP_NAME` matches a registered MCP, exactly.**
- [ ] **`MCP_URL` points at the deployed server, with `/mcp` on the end.**
- [ ] **A setup check runs in CI or on boot**, so a mismatch fails loudly instead
      of quietly disabling every policy.

## Operations

- [ ] **Not on a free tier** if latency matters. Spun-down services take ~30
      seconds to wake, which reads as a hang.
- [ ] **Health checks configured** on both services.
- [ ] **Alerting on the enforcement error rate.** Fail-closed means an ArmorIQ
      outage looks like a very strict policy. You want to know which it is.
- [ ] **Alerting on blocks.** A spike in blocked calls is either an attack or a
      policy that is too tight. Both are worth a page.
- [ ] **The audit trail is monitored, not just collected.** Signed plans are only
      as useful as the attention paid to them.

## Before you trust it

- [ ] **Run it with enforcement disabled once**, and watch what the agent does
      unguarded. If that is not alarming, your tool list may be too timid to be a
      realistic test.
- [ ] **Feed it hostile input on purpose** — a support ticket, document or webhook
      containing instructions — and confirm the enforcement layer holds even when
      the model is fully convinced.
- [ ] **Run the same prompt five times.** Note what varies. Anything your safety
      story depends on must not be in that set.

---

## The one-sentence version

If your agent's safety depends on the model behaving, you do not have a safety
story. Put the decision outside the model, cover every tool that can do harm, and
verify all three outcomes with your own eyes.

---

Back to the **[guide index](./README.md)**
