# NOOS Deliberation Harness — Browser Runtime Observation Contract v0

> Status: Working Design Note / Not a frozen Review Candidate
>
> Purpose: define how the Chrome Shuttle should observe ChatGPT Web execution carriers without becoming a semantic supervisor and without binding product semantics to fragile DOM selectors.

## 1. Core principle

The Shuttle should answer operational questions only:

- is this carrier attached and controllable?
- is the Agent currently generating?
- has the page stopped changing enough to attempt another action?
- is the carrier suspended/discarded/broken?

It should not infer whether the Agent's reasoning is correct, aligned, complete, or ready for Review.

Provider-specific DOM details belong in a replaceable adapter. Product semantics should depend on abstract probes, not CSS class names.

## 2. Observation model: probes, not selectors

A provider adapter may expose probes such as:

```text
composer_present
composer_interactive
stop_generation_control_present
assistant_output_mutating
assistant_message_count
last_assistant_message_stable
route_stable
conversation_identity_resolved
tab_discarded_or_frozen
provider_error_surface_present
```

Each probe may internally use one or more DOM selectors, accessible labels, attributes, MutationObserver signals, URL/navigation signals, or browser tab metadata.

Selectors are implementation details and may change independently from the Harness state machine.

## 3. Do not use a single readiness signal

Examples of unsafe rules:

```text
send_button_visible => READY
stop_button_missing => READY
no DOM mutation for 200ms => READY
```

Any one of these can be false-positive during streaming, post-processing, navigation, citation rendering, fork bootstrap, or provider UI changes.

READY should require a small evidence conjunction rather than one selector.

## 4. GENERATING detection

Strong positive evidence may include:

- the Harness just submitted a prompt and has not observed a completed turn;
- a provider "stop generation" control is present;
- the current assistant response subtree is receiving repeated text/child mutations;
- the composer is intentionally disabled while generation is active.

A generation heartbeat can be maintained from `MutationObserver` activity on the current assistant-output region.

The exact DOM target is provider-specific.

## 5. GENERATING -> STABILIZING

Do not transition directly from "stop button disappeared" to READY.

Enter STABILIZING when strong generation evidence disappears, for example:

```text
stop_generation_control_present == false
AND
assistant_output_mutation_rate has dropped to zero / near-zero
```

The transition should tolerate provider post-processing after textual generation appears finished.

## 6. STABILIZING -> READY

Candidate evidence:

```text
composer_present
AND composer_interactive
AND no strong generation signal
AND last_assistant_message_stable for a quiet window
AND route/conversation identity stable enough for this operation
```

The quiet-window duration is an implementation experiment, not a product contract. A practical initial range may be roughly 1–2 seconds for ordinary continuation, with longer provider-specific stabilization for fork/navigation flows.

READY means only:

> the browser carrier appears operationally safe to receive another Harness action.

It does not imply Logical Control == CONTINUE.

## 7. Separate ordinary turn stabilization from fork stabilization

Ordinary turn:

```text
READY
→ submit prompt
→ GENERATING
→ STABILIZING
→ READY
```

Fork/bootstrap:

```text
Parent READY
→ fork action
→ Child ATTACHING
→ content script attached
→ bootstrap interaction if required
→ provider conversation identity resolves
→ route/title stabilizes
→ optional refresh/rebind validation
→ Child READY
```

Fork READY should be stricter than ordinary turn READY because the new carrier must be re-identifiable after navigation/reload, not merely visually usable once.

## 8. ATTACHING and provider identity

A tab may exist before the provider conversation has durable identity.

ATTACHING should persist until the adapter can establish enough of:

- content script handshake;
- current provider route;
- stable conversation identifier/ref when available;
- usable composer;
- Work Item / Role binding from NOOS.

NOOS must not treat `tabs.onCreated` as equivalent to "worker conversation created".

## 9. SUSPENDED / RECOVERING

Browser-level signals such as discarded/frozen/reloaded tabs are operational, not semantic.

When an active Harness carrier becomes unavailable:

```text
SUSPENDED
→ reactivate/reload/rebind
→ RECOVERING
→ ATTACHING or STABILIZING
→ READY
```

Bindings must survive extension service-worker restart and page reload.

## 10. BROKEN

BROKEN should mean the Shuttle cannot reliably establish the provider adapter contract after bounded recovery attempts.

Examples:

- expected ChatGPT surface cannot be located after reload;
- conversation identity cannot be recovered;
- provider error/interstitial prevents interaction;
- extension/content-script handshake repeatedly fails.

BROKEN is not a semantic failure of the Agent.

## 11. Multi-signal confidence without semantic reasoning

The adapter may internally score evidence or use a small deterministic rule set, but it should not invoke an LLM merely to decide whether the UI is READY.

Example conceptual rule:

```text
GENERATING if any strong generation evidence exists.

READY only if:
  no strong generation evidence
  AND composer is interactable
  AND output is quiet for stabilization window
  AND provider identity/route is sufficiently stable for the intended operation.
```

When evidence is contradictory, prefer STABILIZING/RECOVERING over optimistic READY.

## 12. Mutation observation

`MutationObserver` is appropriate for observing assistant-output changes because it can watch child/text/attribute changes within a DOM subtree without polling the full page.

Observation should be scoped to provider-relevant regions where possible rather than the entire document subtree indefinitely.

DOM mutation activity is evidence of rendering/generation, not semantic content.

## 13. Network interception should not be a v0 dependency

The first Shuttle should not require attaching Chrome DevTools Protocol or intercepting ChatGPT private network APIs merely to determine turn completion.

DOM + browser lifecycle + Harness-known submission events should be sufficient for the first slice.

Network-level instrumentation may later become an optional stronger probe if empirical reliability requires it.

## 14. Provider adapter boundary

Conceptual interface:

```text
ProviderAdapter
  attach(tab)
  observeRuntimeProbes()
  getConversationIdentity()
  isComposerInteractive()
  isGenerationActive()
  observeOutputQuietWindow()
  submit(text)
  fork()
  refreshOrRecover()
```

This is a conceptual product boundary, not an implementation API commitment.

ChatGPT-specific selectors and UI workarounds live behind this adapter.

## 15. Interaction with Logical Control

Browser readiness and workflow permission remain orthogonal:

```text
Carrier Runtime == READY
AND Logical Control == CONTINUE
=> GO permitted
```

Examples:

- `READY + WAIT_REVIEW`: do not GO;
- `GENERATING + CONTINUE`: wait;
- `STABILIZING + CONTINUE`: wait;
- `SUSPENDED + WAIT_REVIEW`: recover carrier if still needed, but do not change the semantic wait state.

## 16. Fail-safe direction

For v0 Step Mode, a false-negative READY mostly costs latency; a false-positive READY can submit into an unstable carrier and cause duplication or corruption.

Therefore the runtime detector should initially bias conservative:

> when uncertain, remain STABILIZING rather than declare READY.

This is an operational fail-safe, not semantic fail-closed reasoning.

## 17. Initial instrumentation / evidence to collect

Before hardening thresholds, log operational telemetry for real manual runs:

- time from visible generation end to stable composer;
- time from last assistant DOM mutation to successful next submission;
- false-positive READY incidents;
- duplicate submit incidents;
- fork tab-created to stable-conversation time;
- refresh/rebind recovery frequency;
- cases where stop-control, mutation heartbeat, and composer state disagree.

The purpose is to tune deterministic browser logic from evidence rather than guess at permanent constants.

## 18. Current working conclusion

The Shuttle should implement a deterministic carrier observer with provider-specific probes and conservative stabilization. It should not encode ChatGPT DOM selectors into higher-level Harness semantics and should not use an LLM to judge browser readiness.

The next design question is the exact submission/idempotency contract: once a carrier is READY, how does NOOS ensure a GO/Review/Sediment command is sent at most once and can recover safely from uncertain browser acknowledgements or extension/service-worker restarts?
