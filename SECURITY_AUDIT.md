# XOLO security review

**Reviewed:** 2026-10-03
**Scope:** The selected `AnshChhetri/Xolo` repository and supplied single-page game; Firebase Realtime Database multiplayer code and rules; third-party script loading; current source and repository history. This is a targeted application review, not a formal penetration test.

## Findings and changes

| Finding | Risk | Change |
| --- | --- | --- |
| Browser dependencies ran without integrity checks, and Tailwind loaded as an executable CDN script. | Supply-chain compromise could execute code in players' browsers. | Added SHA-384 Subresource Integrity to React, React DOM, and Babel. Tailwind's original browser-CDN script was restored unchanged after the local CSS build altered the UI, so that CDN remains a known residual risk. |
| Guests could submit host-level game-control commands to the host processor, and commands used an unbounded push-key queue. | Unauthorized game control and database/host workload abuse. | UI and host processor now reject host-only actions from guests. Rules permit only the six member actions, validate bounded fields, and use one pending command slot per authenticated member. The host also throttles processing to 12 actions per member per 5 seconds. |
| New room invites used six characters. | Guessing risk for newly created rooms. | New invites use eight cryptographically random characters (40 bits from the existing 32-character alphabet). Existing six-character rooms remain joinable for compatibility. |
| Room access needed explicit, deployable server-side boundaries. | Incorrect or permissive production rules could expose room state or allow client writes. | Added `database.rules.json` and `firebase.json`: database-wide access is denied by default; room content is readable only to members; membership creation requires a matching ready invite; only the host may write shared game state; commands are constrained and host-only actions cannot be submitted by guests. |
| Secret exposure was a concern. | A leaked private credential could enable account or service abuse. | Scanned current workspace sources and 19 repository commits for private-key blocks and common cloud, GitHub, Stripe, and Slack credential patterns. No matches were found. The Firebase web `apiKey` is present but is a public client identifier, not a private secret. |

The source scan also found no direct `innerHTML`, `outerHTML`, `dangerouslySetInnerHTML`, `eval`, `new Function`, or `document.write` use in the app code. React escapes rendered text by default.

## Validation

- Firebase Realtime Database rules compiled successfully in the local Emulator Suite.
- **37 rules assertions passed**, including room privacy, host-only writes, valid/invalid command payloads, command-slot limits, identity spoofing rejection, and both current eight-character and legacy six-character invite flows.
- A two-browser app test against local Auth and Database emulators passed room creation, guest join, formation/lineup changes, league setup, the 1v1 kickoff and matchday simulation, with no browser runtime errors.
- A separate Chromium load check confirmed the page mounts without a blank screen or uncaught JavaScript errors. The original Tailwind browser-CDN reference is restored to preserve the existing design.
- A read-only probe of the live database root returned HTTP 401 for an authenticated anonymous test identity; two queried paths were nonexistent and returned `null`. No live room contents were read, and the temporary test identity was deleted.

## Actions still required in Firebase/Google consoles

The production database rules were **not deployed** from this session. Publish the reviewed `database.rules.json` in **Firebase Console → Realtime Database → Rules**, or, after authenticating Firebase CLI and reviewing the target project, run:

```sh
firebase deploy --only database --project projectfootball-bfe26
```

Deploy the updated `index.html` and publish the rules. Also restrict the public Firebase API key to the game's authorized domains and required APIs in Google Cloud Console, and enable Firebase App Check. Those settings require access to the project's consoles and are not changed by this code patch.

## Remaining limitations

Anonymous sign-in remains intentionally enabled, so players still do not need accounts. A room code acts as a bearer invite; legacy six-character rooms retain their lower entropy until those rooms are closed. The host browser remains authoritative for gameplay, so a malicious host can still write game state as the host. The restored Tailwind CDN remains an executable third-party dependency without SRI. App Check and API-key restrictions reduce automated abuse; server-side game validation would require a trusted backend and would be a larger architectural change.

For rule-test setup, see Firebase's [Local Emulator Suite](https://firebase.google.com/docs/emulator-suite) and [Security Rules unit-testing guide](https://firebase.google.com/docs/rules/unit-tests).
