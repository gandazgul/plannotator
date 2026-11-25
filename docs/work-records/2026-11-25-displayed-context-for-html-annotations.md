---
kind: "work_record"
recordId: "f077d31d-59ce-4f32-8579-d7dc87d23477"
status: "approved"
scope: "planned_change"
workKind: "FEATURE"
origin: "internal"
completionMode: "verified"
createdAt: "2026-11-25T02:34:05.632Z"
provenance:
    sourcePlans:
        - "d948b485-18ed-4987-bacf-c26a1199879b"
---
# Displayed context for HTML annotations

## Summary

`HtmlViewer` lets hosts opt in to capture state labels from selected `data-*` attributes on the page root. Each target keeps the labels captured when it was selected, even if the page changes or the target draft reopens. Global comments capture labels when the composer opens and save without context if no valid reply has arrived. Host persistence helpers retain snapshots within the byte budget, and feedback export shows them as escaped labels so reviewers can distinguish page states at the same URL. Stored rows and feedback without displayed context keep the same byte format.

## Out of Scope

This work does not enable capture in Plannotator’s built-in hosts or add state inference, state replay, or support for authoring displayed context through the external annotations API. URL sharing and WebMCP replies do not carry these labels.

## Future Planning Notes

Treat page-provided labels as descriptions, not as proof of state or as input to authorization or revision selection. Keep validation and limits in `@plannotator/core`; omit an invalid or oversized snapshot as a whole rather than truncate a label. When storage reaches its byte limit, drop element context before displayed context, and drop displayed context before removing targets. Publish core before UI when UI depends on a change to this contract.