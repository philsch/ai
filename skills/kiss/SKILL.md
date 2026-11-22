---
name: "kiss"
description: "Keep It Simple, Stupid review. Challenges whether a change or design is the simplest solution to the actual problem and whether a future maintainer will understand it, while never trading away security or stability. Reviews code (current branch diff, PR, or given files) and design documents. Replies with findings and concrete simpler alternatives, leaving the next step to the user. Use whenever the user says kiss, asks if something is over-engineered, too complex, too clever, gold-plated, or asks for a simpler way, a simplicity review, a complexity check, YAGNI check, or wants to know if a design or implementation could be leaner, even if they do not say 'kiss'."
---

# KISS Review

The goal is to find the simplest solution that fully solves the real problem and stays understandable for whoever touches it next. This is not a general code review. Correctness bugs, style, and naming belong to the `code-reviewer` skill. KISS asks one question at two levels: "Is there a simpler way?"

## Ground rules

- **Simple is not the same as short or easy.** Simple means fewer concepts, fewer moving parts, fewer things a reader has to hold in their head at once. A five-line clever one-liner chain can be more complex than fifteen plain lines. Do not suggest code golf.
- **Separate essential from accidental complexity.** Essential complexity comes from the problem itself (regulations, concurrency, money, multi-tenancy). It stays. Accidental complexity comes from the chosen solution. That is what you hunt.
- **Security and stability are load-bearing walls.** Never propose removing or weakening authn/authz, input validation, output encoding, secret handling, tenant isolation, audit logging, error handling on critical paths, timeouts, retries with backoff, idempotency, transactions, data integrity checks, rate limiting, or the observability needed to run the thing in production. You may propose a simpler way to achieve the same protection (for example a framework feature instead of a hand-rolled check), but the protection level must stay equal or improve. If you are not sure whether something is load-bearing, treat it as load-bearing.
- **Consistency is simplicity.** A pattern already used across the codebase costs the reader nothing. A new pattern, library, or service is a new concept to learn, even if it is locally elegant. Weigh that.
- **Do not manufacture findings.** If the solution is already as simple as the problem allows, say so. A short "Simple enough" verdict is a valid and valuable outcome.

## Step 1: Determine the target

Figure out what to review from the user's request:

- **Code, no target given:** run `git fetch origin` and review `git diff origin/main...HEAD` (or `origin/master`). Never diff against local `main`.
- **Code, explicit files or PR:** review those. For a PR, check out or fetch the source branch first.
- **Design document:** read the given Markdown, Confluence page, or draft.

Read the surrounding code too, not only the diff. You cannot judge whether a new abstraction is needed without seeing what already exists in the codebase. Read any `CLAUDE.md` in the repo root and in the changed directories.

## Step 2: Pin down the actual problem

You cannot judge simplicity without knowing what problem is being solved. Collect it from the commit messages, PR description, linked Jira ticket, or the "Problem / Context" section of the design doc. Write it down in one or two sentences, plus the hard constraints (scale, latency, compliance, deadlines, existing contracts).

If the stated problem is vague, note that as the first finding. Most over-engineering starts with solving a broader problem than the one that exists. If the requirement itself is unclear enough that you cannot judge the solution, ask the user before continuing.

## Step 3: Mark the load-bearing parts

Before looking for simplifications, list the parts of the solution that exist for security or stability (see Ground rules). These are off-limits for removal. Keep this list, it goes into the reply so the reader can see what was deliberately protected.

## Step 4: Challenge the approach

Start at the top. A simpler approach beats any number of line-level cleanups.

- Does the problem need solving at all, or does an existing service, platform feature, library, or managed offering already solve it?
- Could this be done with fewer moving parts: fewer services, fewer queues, fewer datastores, fewer deployment units, fewer network hops?
- Is it built for scale, flexibility, or configurability nobody asked for (YAGNI)? Look for "in the future we might..." reasoning.
- Is a synchronous call enough where async messaging was chosen, a cron job enough where a workflow engine was chosen, a DB column enough where a new table or service was chosen?
- Could a smaller first iteration deliver the value, with the rest added when actually needed?
- Does it introduce a new technology or pattern where an established one in the stack would do?

For design documents, also check whether rejected alternatives were rejected for real reasons, and whether the simplest option was considered at all.

## Step 5: Challenge the implementation

Only for code. Typical accidental complexity:

- **Speculative generality:** interfaces with one implementation, factories producing one type, strategy patterns with one strategy, generic type parameters used with one type, plugin systems with one plugin.
- **Indirection without payoff:** wrappers that only delegate, layers that pass data through unchanged, events where a direct call would do.
- **Configuration for constants:** config files, env vars, or feature flags for values that never change.
- **Premature optimization:** caches, pooling, batching, or custom data structures without a measured need.
- **Reinvented wheels:** hand-written versions of what the language, framework, or an already-used library provides.
- **Clever code:** dense functional chains, metaprogramming, reflection, regex where a plain loop or string method is clearer.

## Step 6: Reply to the user

Answer directly in the conversation. Do not write a report file, post PR comments, or change code.

Structure the reply as:

1. **Verdict:** one line, either "Simple enough" or "Simpler options exist".
2. **Problem:** the one- or two-sentence problem statement and hard constraints from Step 2.
3. **Load-bearing parts:** the security and stability parts from Step 3 that were deliberately left alone.
4. **Findings:** approach-level findings first, then implementation-level ones. For each, give the location (`file:line` or doc section), what is more complex than needed, the concrete simpler alternative, and what it removes (concepts, moving parts, lines). Number them so the user can refer to them.

Keep it short. Skip empty sections, and do not pad a "Simple enough" verdict with minor nitpicks.

Then stop. What happens next, such as applying fixes or commenting on a PR, is the user's decision.
