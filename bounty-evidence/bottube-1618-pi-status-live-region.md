# BoTTube accessibility report — Pi auth/payment status is not announced

Bounty: `Scottcjn/rustchain-bounties#1618`  
Claimant / payout route: `woahwhattheheck`  
Checked source: `Scottcjn/bottube@00973f5b3d2098ad42404afb0cce3945d1eead85`

## Finding

The current BoTTube Pi page updates sign-in and payment status asynchronously, but the two status targets are plain `<span>` elements with no live-region semantics. Screen-reader users can therefore miss progress, failure, cancellation, and completion messages that are visible on screen.

Affected source paths at the exact commit above:

- `bottube_templates/pi_home.html`
- `bottube_static/pi_pay.js`

### Sign-in status

`bottube_templates/pi_home.html` defines:

```html
<span class="pi-status" id="pi-status">Open this page in the Pi Browser to sign in.</span>
```

`piSignInUI()` and the `pi:authenticated` event later replace its text with states including:

- `Connecting to Pi…`
- SDK/script-not-ready errors
- timeout/failure messages
- `Signed in as ...`

The flow can also disable and later re-enable the sign-in button while it waits.

### Payment status

The same template defines:

```html
<span class="pi-status" id="pi-setup-status"></span>
```

`bottube_static/pi_pay.js::_payStatus()` repeatedly changes that element while a Pi payment moves through states including:

- `Authorizing with Pi…`
- `Opening Pi payment ...`
- `Approving payment…`
- `Finalizing payment…`
- payment complete
- payment cancelled
- approve/complete/payment errors
- incomplete-payment resume/failure

Neither `#pi-status` nor `#pi-setup-status` has `role="status"`, `aria-live`, `aria-atomic`, or an equivalent status-message mechanism, and the scripts do not move focus to the changed text.

## Accessibility impact

A screen-reader user can activate Pi sign-in or payment and receive no automatic announcement as the operation progresses or fails, even though sighted users see the text change. That makes an asynchronous account/payment workflow materially harder to operate non-visually.

WCAG relevance: **4.1.3 Status Messages (Level AA)** — status information that appears without a context change should be programmatically determinable so assistive technology can present it without focus movement.

## Suggested repair

Use concise status regions for these updates, for example:

```html
<span id="pi-status" class="pi-status" role="status" aria-live="polite" aria-atomic="true">...</span>
<span id="pi-setup-status" class="pi-status" role="status" aria-live="polite" aria-atomic="true"></span>
```

Reserve an assertive alert for errors only if an interruption is genuinely necessary; ordinary progress/success updates can remain polite. Existing button semantics should remain unchanged.

## Duplicate check

Immediately before publishing this report, the full current `#1618` comment history was searched for `pi-status` and `Pi Browser`; no matching report was present. A fresh `Scottcjn/bottube` issue search for `pi-status accessibility` also returned no matching issue. Slack coordination/delegation searches for this exact lane were clean before the lane was claimed.

## Submission receipts

The sponsor's current `docs/HOW_TO_SUBMIT_A_BOUNTY.md` explicitly documents claimant-controlled public report publication as a fallback when a GitHub App receives `403 Resource not accessible by integration` on the sponsor repository.

Direct `/claim` on `Scottcjn/rustchain-bounties#1618`: **403 Resource not accessible by integration**.  
Direct complete report comment on `Scottcjn/rustchain-bounties#1618`: **403 Resource not accessible by integration**.  
Immediate issue-comment readback after the failed write found no `pi-status` comment, so no ghost submission is being claimed.

Claim requested: **1 RTC**, subject to maintainer validation. No native RTC address is being invented; use the registered contributor/GitHub route for `woahwhattheheck` if accepted.
