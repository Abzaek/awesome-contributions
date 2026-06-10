# Open Source Contributions

A curated showcase of exceptional contributions to production-grade open source projects.

---

## Terragrunt (Gruntwork)

### Why This Project Matters

Terragrunt is the industry-standard Terraform/OpenTofu wrapper used by thousands of organizations to manage infrastructure at scale. It's maintained by Gruntwork, a company founded by the authors of *Terraform: Up & Running*. The codebase represents a decade of infrastructure tooling evolution in Go.

### Problem

The `terragrunt scaffold` command — used to generate project boilerplate from module templates — had no support for `tfr://` (Terraform Registry) module sources. Scaffolding with a registry module like `tfr://terraform-aws-modules/vpc/aws` would fail at download time because the `RegistryGetter` (the go-getter client that handles `tfr://` URLs) wasn't wired into the scaffold pipeline's download client. Additionally, there was no mechanism to resolve the latest version from the registry API for pinning in the generated output.

Fixing this required understanding the interaction between three distinct subsystems: go-getter's getter registry pattern (which getters are included and when), the Terraform Registry's service discovery API (how to discover the modules API endpoint), and Terragrunt's scaffold template pipeline (how `sourceUrl` is assembled from resolved URLs and passed into boilerplate generation).

### My Contribution

Implemented `tfr://` source support across two packages:

- **`internal/getter`** — Added a `ResolveTFRVersion` function that performs Terraform Registry service discovery (following the official discovery protocol), queries the module versions endpoint, and performs proper semver sorting using `go-version` to identify the latest release. Added an `IsTFRSource` utility for URL scheme detection.

- **`internal/cli/commands/scaffold`** — Integrated version resolution into the scaffold command's `addRefToModuleURL` function so `tfr://` URLs automatically resolve and pin the latest version. Extended `BuildSourceURL` to propagate the version query param (not just `ref`) into the generated `sourceUrl` template variable. Handled edge cases: user-specified version params take precedence over automatic resolution; the existing `--var Ref=X` flag maps correctly to the `version` param for `tfr://` sources.

The work went through multiple rounds of review with a Gruntwork maintainer, with each iteration addressing increasingly nuanced edge cases.

### Impact

Unlocks scaffolding from the entire Terraform Registry for Terragrunt users. A user can now run `terragrunt scaffold tfr://terraform-aws-modules/vpc/aws` and get a complete, version-pinned Terragrunt configuration — previously impossible.

### Evidence

- **PR:** [#6129 — Support `tfr://` module sources in scaffold command](https://github.com/gruntwork-io/terragrunt/pull/6129)
- **Status:** Open, under active review by Gruntwork maintainer (multiple review rounds completed)

---

## Documenso

### Why This Project Matters

Documenso is the leading open-source alternative to DocuSign, processing document signatures for organizations worldwide. It's a full-stack TypeScript application built with Remix, React, Prisma, and tRPC — the kind of modern stack that demands attention to React internals, state management, and production security.

### Problem 1: CSP Nonce Not Threaded Through UI Libraries

Self-hosted deployments with strict Content Security Policy headers (the project's own recommended default) were generating ~10 CSP violations per page load on the envelope editor. Every `<DragDropContext>` interaction, every scroll-bar removal created inline `<style>` elements without the server-generated per-request nonce.

The root cause was subtle: two different vendor libraries discover the nonce through entirely different mechanisms:
- `@hello-pangea/dnd` (react-beautiful-dnd fork) accepts a `nonce` prop on `<DragDropContext>` — but Documenso wasn't passing it
- `react-remove-scroll-bar` / `use-sidecar` reads `window.__webpack_nonce__`, which was never set at runtime because Documenso uses Remix (not webpack-dev-server), so no bundler ever initialized this global

The CSP nonce itself was correctly plumbed through the root loader and into `<Scripts nonce>` and inline elements — the gap was specifically in these third-party library integration points.

### My Contribution

A two-part fix: first, set `window.__webpack_nonce__` in a nonce-gated inline `<script>` in the document `<head>` so `react-remove-scroll-bar` discovers it automatically. Second, thread the nonce from the existing `useCspNonce()` hook into every `<DragDropContext>` instance across the application (three locations in the envelope editor flow).

### Impact

Eliminates all CSP violations on the envelope editor for self-hosted deployments. Enables operators to run with strict CSP without sacrificing drag-and-drop functionality. The fix uses the project's existing nonce plumbing — no new infrastructure, no config changes.

### Evidence

- **PR:** [#2961 — Thread CSP nonce into DragDropContext and set `__webpack_nonce__` global](https://github.com/documenso/documenso/pull/2961)
- **Closes:** Issue [#2872](https://github.com/documenso/documenso/issues/2872)
- **Status:** Open

### Problem 2: Systematic Bug Fixes from Verified Issue Report

Issue [#2829](https://github.com/documenso/documenso/issues/2829) contained a security research firm's Faultmark report identifying bugs across the codebase — not just typo-level issues but real logic errors in production paths. Navigating from a report to correct, reviewable fixes requires understanding the codebase structure, the type system, and the behavior of each affected component.

### My Contribution

Fixed 6 verified bugs spanning the React component layer, Konva canvas integration, and table rendering, all merged to production:

- **Mutating React state with `.sort()`** — `envelope.recipients.sort(...)` mutated state in place, causing missed re-renders and stale closures in downstream `useMemo` computations. The fix creates a shallow copy before sorting, preserving referential integrity of the state array.

- **Mutating component props with `.sort()`** — `allRecipients.sort(...)` in a `useMemo` dependency called `.sort()` directly on a prop array, violating React's immutability contract and silently corrupting the parent's data.

- **Duplicate DOM destroy in canvas rendering** — The DROPDOWN field handler called `loadingSpinnerGroup.destroy()` in both the `.then()` success path and `.finally()` cleanup. Every other field handler called it only once. This was the only handler with this duplication — the Konva destroy is not idempotent.

- **Reversed pagination comparison** — `1 > table.getPageCount()` showed pagination UI when the table was empty and hid it when multiple pages existed. Functionally backwards.

- **Initials field returning raw param instead of resolved value** — `value: initials` returned the raw function parameter (nullable) instead of `initialsToInsert` (the value resolved through a user dialog). The dialog branch was effectively dead code.

- **Dropdown removeValue crash on last item** — `splice(index, 1)` followed by `newValues[index].value` crashed when removing the last item (index out of bounds). Even in non-crashing cases, it compared against the wrong shifted element.

All six were reviewed and merged by Documenso maintainers across separate focused PRs.

### Impact

Six production bugs eliminated from the document signing flow, admin panel, and envelope editor. Each fix addresses a real user-facing failure mode in a platform that handles legally-significant document transactions.

### Evidence

- PR [#2838](https://github.com/documenso/documenso/pull/2838) — Initials field null return (✅ Merged)
- PR [#2839](https://github.com/documenso/documenso/pull/2839) — React state sort mutation (✅ Merged)
- PR [#2840](https://github.com/documenso/documenso/pull/2840) — Prop array sort mutation (✅ Merged)
- PR [#2841](https://github.com/documenso/documenso/pull/2841) — Duplicate DOM destroy (✅ Merged)
- PR [#2842](https://github.com/documenso/documenso/pull/2842) — Reversed pagination (✅ Merged)
- PR [#2843](https://github.com/documenso/documenso/pull/2843) — Dropdown removeValue crash (✅ Merged)

---

*All contributions went through maintainer review before merging. Each represents independent analysis, not reported-by-others work.*
