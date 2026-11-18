---
planId: "d78c5598-f231-49eb-9ea0-4a4ed0cf16ab"
classification: "PLANNED_CHANGE"
workKind: "FEATURE"
complexity: "HIGH"
affectedPaths:
  - "packages/ui/components/html-viewer/HtmlViewer.tsx"
  - "packages/ui/components/html-viewer/srcdoc.ts"
  - "packages/ui/components/html-viewer/srcdoc.test.ts"
  - "packages/ui/components/html-viewer/index.ts"
  - "packages/ui/components/html-viewer/HtmlViewer.resourcePolicy.test.tsx"
  - "tests/dom-test-list.txt"
  - "tests/e2e/html-viewer-resource-policy.spec.ts"
  - "packages/ui/README.md"
  - "packages/ui/HANDOFF.md"
  - "docs/agent-guides/guides-and-shared-ui.md"
executionAgent: "frontend-engineer"
collaborationRecommendation: "autonomous"
createdAt: "2026-11-12"
origin: "internal"
userVerifiedAt: null
targetBranch: "main"
deliveryBranch: "plan/plannotator-host-html-restrictions"
status: "implemented"
---

# Host-Controlled Restrictions for HTML Annotation

## Context

A host needs to show self-contained HTML, let people interact with it, and annotate it, without letting the page load
external resources. `HtmlViewer` cannot do this today, for two reasons:

- **Author CSP is removed and no host CSP exists.** `neutralizeMetaCsp()` (`srcdoc.ts`) removes every author
  `<meta http-equiv="Content-Security-Policy">`, so the inline bridge can run. Nothing replaces it. The iframe sandbox
  gives an opaque origin, but it does not stop network requests.
- **Author code can select live bridge mode.** `BRIDGE_SCRIPT` reads `window.__plannotatorLiveConfig` once, at
  `bridge-script.ts` line 294. The bridge loads at the end of `<head>` (`injectIntoHead()`), after author head scripts.
  An author script that sets the global first switches the bridge to live behavior. With `nested: true`, the bridge
  then posts to `parent.parent`, the host window.

The [proposal](plannotator-fork-extensions.md) records the shared decisions for both fork PRs. The
[setup handoff](plannotator-setup-handoff.md) records the verified fork, the upstream base
`f8c1b58bac1c78c4103b4a548ba4f0d6ee908824`, and this worktree: `~/web/plannotator/.worktrees/host-html-restrictions/`
on `feat/host-html-restrictions`. All source paths below are relative to Plannotator. This Plan is not a RunWield
submodule patch. The planning files are excluded through `.git/info/exclude`; do not add them to the PR.

This Plan supplies a library prerequisite for **Preserve a simple, contained review experience** in
[interactive prototype review — external RunWield source](file:///Users/gandazgul/web/runwield/docs/prd/product-planner-prd.md#interactive-prototype-review).
It does not deliver RunWield integration, storage, or workflow. Plannotator has no PRD for `HtmlViewer`; its supported
host contract lives in `packages/ui/README.md` and `packages/ui/HANDOFF.md`, which change in the same PR.

## Objective

`HtmlViewer` accepts an optional host policy, `resourcePolicy="self-contained"`. With it:

- The browser receives a host Content Security Policy (CSP) before any author markup. External scripts, styles,
  images, fonts, frames, and data requests are blocked.
- Inline interaction and real annotations still work.
- Author code cannot select live bridge mode through `window.__plannotatorLiveConfig`.

Without the option, every current caller gets the same output and behavior as today.

## Approach

### A narrow opt-in contract

**Owner decision:** this PR closes only the two gaps above. It reuses the iframe sandbox and adds no general
security-policy framework.

The self-contained profile permits inline scripts, inline event handlers, inline styles, embedded `data:` images, and
the inline annotation bridge. Remote and sibling-file dependencies are not self-contained.

**Reviewable naming assumption:** the prop is `resourcePolicy?: "self-contained"`, exported as the type
`HtmlResourcePolicy`. It selects one documented policy. It is not a page-authored CSP or a configurable allowlist.

The host CSP is one exported constant, `SELF_CONTAINED_CSP`, in `srcdoc.ts`:

```text
default-src 'none'; script-src 'unsafe-inline' 'unsafe-eval'; style-src 'unsafe-inline';
img-src data:; font-src data:; connect-src 'none'; frame-src 'none';
object-src 'none'; worker-src 'none'; base-uri 'none'; form-action 'none'
```

**Reviewable assumption — `font-src data:`:** single-file exports often embed fonts as `data:` URLs. This loads nothing
external. Without it, those pages fall back to system fonts.

**Owner decision — `'unsafe-eval'`:** `script-src` allows eval. It loads nothing external, because inline script is
already allowed. It lets eval-based libraries run, such as Alpine.js or Vue's in-browser template compiler.

### Rejected combinations fail loudly

The restricted frame uses `sandbox="allow-scripts"`: no same-origin, popup, modal, form, download, or top-navigation
allowance. Some current props widen that frame or bypass the srcdoc. With `resourcePolicy`, `HtmlViewer` throws a
descriptive `Error` during render when any of these is present:

| Prop | Why it conflicts |
| --- | --- |
| `src`, `liveSession`, `liveSandbox` | Live mode loads a real origin through the proxy, with no srcdoc. |
| `bridgeScriptUrl` | An external bridge needs a script origin. **Owner decision:** inline bridge only. |
| `frameUrl` | Serves the document under the host's own policy, which exists to allow web pictures. |
| `hostNavigates` (true) | Adds `allow-popups`. |
| `sandbox` other than `"allow-scripts"` | Any other value widens or replaces the restricted sandbox. |

A throw is the existing precedent: `buildBridgeScriptTag()` throws when the inline bridge is stubbed and no URL is given.
The viewer never falls back to unrestricted rendering.

`HtmlViewer` destructures `sandbox` with a default. The check must read whether the caller passed a value, so the
default `"allow-scripts allow-modals"` does not trigger a false rejection.

### The policy goes first in the finished document

The srcdoc pipeline gets one new last step. It runs after every existing helper, so no later helper can strip the host
CSP:

```diff
 srcdoc = injectIntoHead(injectPrintDefaults(rawHtml), injection)   // neutralizes author CSP
 srcdoc = bridgeUrl ? srcdoc : injectSandboxShim(srcdoc)
+srcdoc = resourcePolicy ? applySelfContainedPolicy(srcdoc) : srcdoc
```

`applySelfContainedPolicy(document)` inserts two elements at the document start:

```text
<meta http-equiv="Content-Security-Policy" content="SELF_CONTAINED_CSP">
<script>lock window.__plannotatorLiveConfig to null</script>
```

The document start is directly after a leading doctype, when only whitespace and comments precede that doctype.
Otherwise it is index 0.

> [!TIP]
> **Why the start, not the author's `<head>`**
>
> A meta CSP is honored only inside `<head>`, and only for content after it. The HTML parser puts an element that comes
> before `<html>` into an implied `<head>`. The author's later `<html>` tag adds its attributes to that element, and the
> later `<head>` tag is ignored. So the start position is in `<head>` for every input. The author's `<head>` position is
> not: an `<img>` or text before `<head>` opens the body, and a meta CSP there is ignored. Inserting after the doctype
> keeps standards mode.

This needs no head search. `headStart()` is not reused for this step: it can stop after body content has started.

The lock script runs before every author script:

```js
Object.defineProperty(window, "__plannotatorLiveConfig", {
  value: null, writable: false, configurable: false, enumerable: false,
});
```

The bridge reads the global once, so it always sees `null` and runs srcdoc behavior. Assignment fails, and
`defineProperty` and `delete` cannot change a non-configurable property. `bridge-script.ts` does not change, so the Pi
vendored copy does not change either.

**Set aside:** locking the global in every srcdoc document would also close the hole for unconfigured hosts. It changes
default output, and the owner kept unconfigured behavior unchanged.

### Reload when the policy changes

The iframe gets a `key` derived from `resourcePolicy` (a constant for unconfigured callers). Changing the policy mounts
a new iframe, so the new sandbox and CSP apply to a new document load. The current document is never relabeled.

```text
<HtmlViewer resourcePolicy> (HtmlViewer.tsx)
  assertResourcePolicyProps(props)        // throws on a rejected combination
  srcdoc = … → applySelfContainedPolicy   // last step
  <iframe key={policy} srcDoc sandbox="allow-scripts">
```

## Expected Change Surface

The boundaries this change is expected to touch. This list is guidance, not an allowlist: verify the real footprint
during implementation and change whatever the Implementation Steps need, including files not named here. Stop and report
only when discovery changes approved intent — the change reaches another subsystem, public behavior or architecture
shifts, migration or compatibility risk grows, or the Verification Plan no longer proves the objective.

- `packages/ui/components/html-viewer/srcdoc.ts` — `SELF_CONTAINED_CSP`, the document-start finder, and
  `applySelfContainedPolicy()`.
- `packages/ui/components/html-viewer/HtmlViewer.tsx` — the `resourcePolicy` prop, rejected combinations, the last
  pipeline step, the forced sandbox, and the iframe key.
- `packages/ui/components/html-viewer/index.ts` — exports the `HtmlResourcePolicy` type.
- `srcdoc.test.ts` — insertion position and lock-script content for each input shape.
- `HtmlViewer.resourcePolicy.test.tsx` (new, DOM-gated) and `tests/dom-test-list.txt` — prop validation, sandbox,
  srcdoc, and remount. The DOM allowlist guard fails if the file is not listed.
- `tests/e2e/html-viewer-resource-policy.spec.ts` (new) and a small harness entry beside it — the real browser proof.
  It runs in the existing `tests/e2e/playwright.config.ts` Chromium project with no config change.
- `packages/ui/README.md`, `packages/ui/HANDOFF.md` — the supported host contract.
- `docs/agent-guides/guides-and-shared-ui.md` — the raw-HTML annotate notes that describe CSP removal and the sandbox.

Not in this PR: `bridge-script.ts`, the Pi vendor copies, server routes, the live proxy, the Artifact Server, RunWield
source, dependency pins, and annotation context work.

## Reuse Opportunities

- `injectIntoHead`, `injectPrintDefaults`, `injectSandboxShim`, `buildSrcdocInjection` — the unchanged pipeline that the
  new step follows.
- `buildBridgeScriptTag()` — the precedent for throwing on an unusable configuration.
- `HtmlViewer.sandbox.test.tsx` — its `frameFor()` mount pattern for the new DOM test.
- `tests/e2e/html-print-defaults.spec.ts` — a no-build Chromium spec in the existing config.
- `tests/e2e/surface.spec.ts` — its method for proving a refused `fetch` in a real browser. Do not copy its restrictive
  parent CSP: here the parent page has no CSP, so the viewer option alone must block requests.

## Implementation Steps

1. **`srcdoc.ts` owns the policy.** It exports `SELF_CONTAINED_CSP` (exactly the CSP above) and `applySelfContainedPolicy(document: string): string`. The function inserts the CSP `<meta>` and the lock
   `<script>` directly after a doctype that only whitespace and comments precede, and otherwise at index 0. The doctype
   match is case-insensitive. It never removes or edits any other markup.
2. **`HtmlViewer` accepts and validates the option.** `HtmlViewerProps.resourcePolicy?: HtmlResourcePolicy` is
   documented in the prop's JSDoc. With the option, each rejected combination in the Approach table throws an `Error`
   whose message names the conflicting prop. A `sandbox` prop that the caller did not pass is not a conflict.
3. **The restricted srcdoc is correct.** With the option, the srcdoc memo applies `applySelfContainedPolicy` as its
   last step, and the iframe `sandbox` attribute is exactly `allow-scripts`. Without the option, the srcdoc string and
   the sandbox attribute are byte-identical to the current output.
4. **A policy change reloads the frame.** The iframe `key` derives from `resourcePolicy`. Toggling the option mounts a
   new iframe element with the new sandbox and srcdoc. Unconfigured callers keep one stable key.
5. **Unit and DOM tests guard the contract.** `srcdoc.test.ts` covers each input shape listed in the Verification Plan.
   `HtmlViewer.resourcePolicy.test.tsx` covers each rejected prop, the forced sandbox, the srcdoc start, the remount,
   and the unchanged default.
6. **A browser spec proves enforcement and usability together.** `tests/e2e/html-viewer-resource-policy.spec.ts` mounts
   the real `HtmlViewer` in Chromium and runs the hostile fixture with and without the option, as the Verification
   Plan specifies. The harness can also run standalone for the headed check, and it prints its URL.
7. **Host documentation matches the delivery.** `packages/ui/README.md` (the `HtmlViewer` section) and a new
   `packages/ui/HANDOFF.md` section document: the prop and an example; the exact CSP; the rejected combinations; that a
   stricter host page CSP is inherited and can block the inline bridge; that `<base href>` is ignored; and that
   navigation is not blocked. `docs/agent-guides/guides-and-shared-ui.md` states that the opt-in policy is the one case
   where the viewer adds a CSP.

## Approval Confirmation

No Work Records are proposed for supersession. Fork, clone, and worktree setup is complete (see the setup handoff). The
owner selected the self-contained profile with the inline bridge and `'unsafe-eval'`, and limited this PR to early resource restrictions
and non-live bridge enforcement. The prop name and `font-src data:` are reviewable assumptions. No RunWield dependency
upgrade is authorized.

## Verification Plan

Run from this worktree with Bun 1.4.4. Install first with `bun install --frozen-lockfile`; the worktree has no
`node_modules`. Run focused tests through `bun run test:sandbox`, which isolates home and data folders and routes DOM
and e2e files to the correct runner.

**Automated, focused:**

- `bun run test:sandbox packages/ui/components/html-viewer/srcdoc.test.ts packages/ui/components/html-viewer/sandboxShim.test.ts packages/ui/components/html-viewer/bridgeAsset.test.ts`
- `bun run test:sandbox packages/ui/components/html-viewer/HtmlViewer.resourcePolicy.test.tsx packages/ui/components/html-viewer/HtmlViewer.sandbox.test.tsx packages/ui/components/html-viewer/HtmlViewer.bridgeAsset.test.tsx packages/ui/components/html-viewer/HtmlViewer.frameUrl.test.tsx packages/ui/components/html-viewer/htmlLiveProtocol.test.tsx packages/ui/components/html-viewer/htmlLinkProtocol.test.tsx packages/ui/components/html-viewer/htmlPinpointProtocol.test.tsx`
- `bun run test:sandbox tests/e2e/html-viewer-resource-policy.spec.ts tests/e2e/html-print-defaults.spec.ts`
- `bun run test:sandbox scripts/dom-test-allowlist.test.ts`
- `bun run --cwd packages/ui typecheck` and `bun run --cwd packages/ui smoke:package`

**`srcdoc.test.ts` — placement.** For each input, the CSP `<meta>` is the first element, followed by the lock script:
doctype with head; uppercase `<!DOCTYPE>`; comments before the doctype; no doctype; no `<head>`; a `<script src>` or
`<img>` before `<head>`; fake `<head>` text inside a comment, a `<script>`, or an attribute value; an author meta CSP;
and a fragment. The rest of the document is unchanged byte for byte. A version that inserts at the author's `<head>`
must fail the `<img>`-before-`<head>` case.

**`HtmlViewer.resourcePolicy.test.tsx` — the component contract.**

- Each prop in the rejected-combinations table throws and mounts no iframe.
- With the option, the sandbox is exactly `allow-scripts`, and the srcdoc starts with the host CSP.
- Toggling the option replaces the iframe element (the element reference changes).
- Without the option, the srcdoc and sandbox match the current output for the same input.

**`tests/e2e/html-viewer-resource-policy.spec.ts` — the browser proof.**

- **Harness.** The spec bundles a small entry that mounts the real `HtmlViewer` (`bun build --target browser`; Bun is in
  the CI image). A loopback server serves the harness page with no CSP. A second loopback server, on another port,
  counts every request by path and returns valid responses. No test touches `~/.plannotator`.
- **Hostile fixture.** The page tries each resource class early (in markup before `<head>`, and in `<head>`) and late
  (inserted by script after load): script, stylesheet, CSS `@import`, CSS `url()` image, `<img>`, font, nested
  `<iframe>`, `fetch`, `XMLHttpRequest`, `WebSocket`, `navigator.sendBeacon`, and a worker.
- **Control run (no option).** Each request class reaches the counting server. This proves the endpoints work, so zero
  counts in the restricted run mean the requests were blocked. A browser refuses a cross-origin worker in both runs, so
  the restricted run proves the worker class with a `securitypolicyviolation` event instead of a count.
- **Restricted run.** The counting server receives zero requests, and the frame reports CSP violations for the covered
  classes. Author `<html>` attributes survive, and the page is in standards mode.
- **Usable under restriction, same run.** Click an inline `onclick` handler and an `addEventListener` button that update
  a simulated cart count. Run one `new Function` call and show its result. See an embedded `data:` image. Select text and submit a comment. Make a pinpoint comment and
  submit it. Reload with those annotations and see the highlights restored. Assert the `onAddAnnotation` payloads
  (quote, anchor), not only a bridge-ready event. A blanket script block, a stub bridge, or a wrong CSP that blocks
  inline styles must fail this part.
- **Live-config lock.** A head script before the bridge assigns `window.__plannotatorLiveConfig = { nested: true, … }`,
  then tries `delete` and `Object.defineProperty`; a later script repeats this after the bridge loads. The global
  stays `null`, and text and pinpoint comments still submit. Without the lock, `nested: true` makes the bridge's frame
  gate exit, so no comment can submit: this case fails if the lock is missing or runs after author scripts.
- **Inherited host CSP.** A separate case serves the harness page with `script-src 'self'`. Record what happens: the
  inline bridge is blocked and annotation does not start. This proves the documented limitation; it is not a pass/fail
  claim about loosening the policy.

**Behavior that existing tests must still protect.** Default sandbox `allow-scripts allow-modals`; `sandbox` and
`hostNavigates` behavior; the `bridgeScriptUrl` timeout and version-mismatch banner; `frameUrl` serving; live mode and
the live config in proxied pages; link relaying and pinpoint protocol; theme neutrality; print defaults; and the sandbox
shim position. No existing test is expected to stop applying. If one no longer compiles, rewrite it against the new
shape; do not delete it.

**Headed check (frontend Engineer).** Start the harness standalone and open its printed URL with
`agent-browser --headed --session host-html-restrictions`. With the option on: change the cart state, select text,
make a pinpoint comment, and submit. Inspect the console for CSP violation reports and confirm the counting server
shows zero requests. Record the exact harness command and URL in the handoff.

**AI review.** Inspect every path that sets the iframe `src` or `srcDoc`. Confirm the policy step is last, that every
read of `__plannotatorLiveConfig` happens after the lock, and that the README makes no navigation or no-network
guarantee.

## Edge Cases & Considerations

- **Navigation is not blocked.** CSP has no navigation directive. An author can set `location`, use a `<meta
  http-equiv="refresh">`, or follow a link the bridge does not catch, and that request reaches the network. The sandbox
  still blocks top-level navigation and popups. Document this; do not claim a no-network guarantee.
- **The page shares a realm with the bridge.** This change restricts resource loading and live-mode selection. It does
  not stop author code from tampering with the bridge or forging annotation text.
- **A stricter host page CSP is inherited.** A srcdoc frame inherits the host page's CSP, and the policies intersect. A
  host that forbids inline scripts blocks the inline bridge. The inline path has no ready timeout, so no banner shows.
  Document the limitation; do not add a new failure banner.
- **`base-uri 'none'` ignores `<base href>`.** Hosts that rewrite asset paths with a `<base>` (annotate's
  `rewriteHtmlAssetReferences`) get no benefit under this policy, because asset loads are blocked anyway. Link relaying
  still reports the raw `href`.
- **`blob:` images are blocked.** A page that draws a canvas and shows it through an object URL breaks. Add `blob:` to
  `img-src` only if the owner asks; it loads nothing external.
- **The doctype rule protects standards mode.** A doctype that comes after other content already puts the page in
  quirks mode, so inserting at index 0 changes nothing there.
- **The bridge-asset alias is incompatible.** A host that aliases `bridge-script` to the lite module has no inline
  bridge. With the option, `buildBridgeScriptTag()` throws its existing error, which is correct.
