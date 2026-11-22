---
kind: "work_record"
recordId: "cf756079-3866-4341-b13a-4ab02f8ad79a"
status: "pending_verification"
scope: "planned_change"
workKind: "FEATURE"
origin: "internal"
completionMode: "verified"
createdAt: "2026-11-22T22:23:43.740Z"
provenance:
    sourcePlans:
        - "d78c5598-f231-49eb-9ea0-4a4ed0cf16ab"
---
# Host-controlled resource restrictions for HTML annotation

## Summary

`HtmlViewer` offers an opt-in `resourcePolicy="self-contained"` for reviewing interactive, single-file HTML. A host Content Security Policy blocks remote and sibling-file resources, nested frames, workers, and network data requests while allowing inline scripts and styles, embedded `data:` images and fonts, and the inline bridge for text and pinpoint annotations. The viewer uses an `allow-scripts`-only sandbox and prevents page scripts from selecting live bridge mode. Conflicting props cause a render error rather than an unrestricted fallback, and callers that omit the option keep the existing behavior.

## Out of Scope

This work does not include RunWield integration, annotation storage or workflow, or a configurable resource allowlist.

## Future Planning Notes

Apply the host policy after all other `srcdoc` processing, but place it and the live-mode lock before author markup. Keep the bridge inline and replace the iframe when the policy changes. A stricter host page policy still applies and can block the inline bridge. The browser ignores `<base href>`, but the iframe can still navigate itself and make a network request. Author scripts share the bridge's execution environment, so this option does not prevent bridge tampering or forged annotation text.