# Contributing

This project is documentation that an agent executes. That makes contributions unusual:
there is nothing to compile, and "does it work" means "does an agent, reading only this,
build the right thing in a repo it has never seen."

## Most valuable contributions

1. **Worked mappings for a new stack.** `references/02-data-model.md` ends with the same
   model expressed in Convex, Prisma, and a document store. Adding Drizzle, TypeORM,
   ActiveRecord, Django, or Laravel — with the real index syntax and the real gotchas —
   directly widens who this works for.
2. **Troubleshooting entries.** `references/08-troubleshooting.md` is the highest-leverage
   file in the repo. If an install went sideways in your codebase, the symptom → cause pair
   belongs there. Real failures beat imagined ones.
3. **Verification cases.** If you found a bug the 20 backend cases in
   `references/07-verification.md` would have missed, add the case.
4. **Repo shapes that need a different plan** — mobile, monorepo, static-site, GraphQL,
   per-tenant branding. Say what changes and what stays the same.

## Writing style for references

The audience is an agent with limited context, working in an unfamiliar repository.

- **Contracts, not code.** Specify semantics, arguments, return shapes, and failure modes.
  Illustrative snippets are welcome; copy-paste-ready framework code is not — it will be
  wrong in most repos and worse, it will be *pasted*.
- **State the why for anything counter-intuitive.** Every unexplained rule gets
  "simplified" away by the next contributor or the next agent. The receipts-vs-watermark
  argument in `02-data-model.md` is the model to follow.
- **Prefer a rule that fails loudly.** "Reject empty alt text at publish" beats "remember
  to add alt text."
- **Mark optional work `[Optional]`.** Scope creep in a spec becomes scope creep in every
  install.
- **No hedging, no filler.** An agent reading "you may want to consider possibly" learns
  nothing. Say what to do.
- Keep each reference independently loadable — an agent may read one without the others.

## Changing an invariant

The nine invariants in `SKILL.md` are load-bearing; several of them exist because the
obvious implementation is subtly wrong (per-user dismissal, no-backfill, server-side
audience filtering). Changing one needs a written rationale in the PR and a matching update
to the verification cases that test it.

## Testing a change

There is no unit test suite. The real test is an end-to-end install:

1. Pick a repository the change is meant to help — ideally one you did not write.
2. Run `/changelog:install` with the modified plugin.
3. Run `/changelog:verify` and read the report critically. Did the agent make the decision
   you intended? Did it produce code that looks native to that repo?
4. Note in the PR which stack you tested against, and what the agent got wrong before your
   change.

A change tested against one real repo is worth more than a change that reads well.

## Pull requests

- One concern per PR.
- Say which stacks you verified against.
- If you added a `[Optional]` feature, explain why it does not belong in the default build.
