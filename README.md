# Open Source Contributions

A curated showcase of my open source contributions — real bugs fixed, features added, and projects improved.

---

## Documenso — Open Source DocuSign Alternative

**Tech Stack:** TypeScript, React, Remix, Prisma, PostgreSQL, TRPC

Documenso is the leading open-source document signing platform. I contributed multiple bug fixes validated against their verified issue tracker, all of which have been reviewed and merged by maintainers.

### Merged PRs

**1. React State Mutation: `.sort()` on State Array**
- **Problem:** `envelope.recipients.sort(...)` mutates React state directly, causing stale closures, missed re-renders, and incorrect `selectedAssistantRecipient` behavior on subsequent renders.
- **PR:** [#2839](https://github.com/documenso/documenso/pull/2839)
- **Fix:** Spread the array before sorting: `[...envelope.recipients].sort(...)`

**2. Prop Array Mutation: `.sort()` on Component Props**
- **Problem:** `allRecipients.sort(...)` mutates the prop array received from a parent component, violating React's immutability contract and causing unpredictable behavior in the parent.
- **PR:** [#2840](https://github.com/documenso/documenso/pull/2840)
- **Fix:** Spread the prop before sorting: `[...allRecipients].sort(...)`

**3. Duplicate DOM Destroy Call**
- **Problem:** `loadingSpinnerGroup.destroy()` called in both `.then()` and `.finally()` blocks, causing a double-destroy on the DROPDOWN sign path. Every other field handler called `destroy()` only once in `.finally()`.
- **PR:** [#2841](https://github.com/documenso/documenso/pull/2841)
- **Fix:** Removed the redundant `destroy()` in `.then()`, matching the pattern used by all other handlers.

**4. Reversed Pagination Comparison**
- **Problem:** `1 > table.getPageCount()` in the admin organisations table was a logic error — it showed pagination for empty tables (`pageCount === 0`) but hid it when multiple pages existed.
- **PR:** [#2842](https://github.com/documenso/documenso/pull/2842)
- **Fix:** Reversed to `table.getPageCount() > 1`.

**5. Initials Field Returning Null Instead of User Input**
- **Problem:** `handleInitialsFieldClick` returned `value: initials` (the raw param, possibly null) instead of `value: initialsToInsert` (the resolved value after dialog prompt). User-entered initials were discarded.
- **PR:** [#2838](https://github.com/documenso/documenso/pull/2838)
- **Fix:** Changed return value to `initialsToInsert`.

**6. Dropdown `removeValue` Crash on Last Item**
- **Problem:** `newValues.splice(index, 1)` followed by `newValues[index].value` crashes when the removed item was the last in the array (undefined access). Had a secondary logic bug — even when it didn't crash, it compared against the wrong shifted element.
- **PR:** [#2843](https://github.com/documenso/documenso/pull/2843)
- **Fix:** Capture `removedValue = currentValues[index].value` before splice, compare against the actually removed value.

**7. CSP Nonce Threading into DragDropContext** *(in review)*
- **Problem:** `@hello-pangea/dnd` DragDropContext doesn't thread Content-Security-Policy nonces through to its internally created `<style>` and `<script>` elements, causing CSP violations in production with strict Content-Security-Policy headers.
- **PR:** [#2961](https://github.com/documenso/documenso/pull/2961)
- **Fix:** Retrieve the CSP nonce from document metadata and inject it via `__webpack_nonce__` and custom DragDropContext configuration.

**8. Session Cookie Expiry Evaluation at Module-Load Time**
- **Problem:** Session cookie expiry was evaluated once at module-load time rather than at request time, causing stale expiration checks.
- **PR:** [#2960](https://github.com/documenso/documenso/pull/2960)
- **Fix:** Convert expiry evaluation to a lazy/thunk pattern so it runs per-request.

---

## Terragrunt — Infrastructure as Code Tool by Gruntwork

**Tech Stack:** Go, Terraform/OpenTofu, go-getter

Terragrunt is a thin wrapper for Terraform/OpenTofu that provides extra tools for keeping configurations DRY, working with modules, and managing remote state.

### Open PR

**9. Support `tfr://` Module Sources in Scaffold Command**
- **Problem:** `terragrunt scaffold` failed when using `tfr://` (Terraform Registry) module sources — the `RegistryGetter` wasn't wired into the download client, and the scaffold command couldn't resolve the latest version from the registry API.
- **PR:** [#6129](https://github.com/gruntwork-io/terragrunt/pull/6129)
- **Impact:** Enables scaffolding from the Terraform Registry, adding a new source type to Terragrunt's scaffold workflow. Changes span two packages: the getter (`internal/getter`) and scaffold command (`internal/cli/commands/scaffold`).

---

## Hatchet — Open Source Workflow Engine

**Tech Stack:** Go, PostgreSQL, Temporal

Hatchet is a distributed, fault-tolerant workflow orchestration engine.

### Comment

**10. Database Connection Pool `statement_timeout` Leak** *(under review)*
- **Problem:** PostgreSQL `statement_timeout` settings set via `SET` in one query leak across pooled database connections, causing subsequent queries in the same connection to inherit unintended timeout configurations.
- **Issue:** [#3898](https://github.com/hatchet-dev/hatchet/issues/3898)
- **Proposed Fix:** Replace `SET statement_timeout` with `SET LOCAL statement_timeout` to scope configuration to the current transaction, and add pool connection initialization to reset session state on reuse.

---

*This repository showcases contributions to production-grade open source projects. All changes went through code review by maintainers.*
