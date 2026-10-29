# Domain model — User context

- **Status**: Implemented
- **Last updated**: 2026-07-27
- **Source**: `ui-ui-color-palette/src/types/{app,config,user}.ts`, `src/bridges/checks/`, `src/bridges/plans/enableTrial.ts`, `src/stores/{consent,credits,preferences}.ts`

## 1. Identity and session

```
UserSession
├─ connectionStatus: 'CONNECTED' | 'UNCONNECTED'
├─ userId: string
├─ userFullName: string
└─ userAvatar: string

Identity
├─ connectionStatus
├─ userId?: string
└─ creatorId: string      # distinguishes the current user from a palette's creator (case of a palette published by someone else)
```

## 2. Commercial model types

```
PlanStatus   = 'UNPAID' | 'PAID' | 'NOT_SUPPORTED'
TrialStatus  = 'UNUSED' | 'PENDING' | 'EXPIRED' | 'SUSPENDED'
```

`BaseProps` (context injected into the UI) carries `planStatus`, `trialStatus`, `trialRemainingTime`, `creditsCount`, `creditsRenewalDate`.

`Config.plan` defines the global parameters: `isProEnabled`, `isTrialEnabled`, `isCreditsEnabled`, `trialTime`, `creditsLimit`, `creditsRenewalPeriodDays`, plus store IDs (`storeProWeekId`, `storeProMonthId`, `storeProYearId`, `storeProLifetimeId`).

## 3. Checks (`src/bridges/checks/`)

| Check | Likely role |
| --- | --- |
| `checkAnnouncementsStatus` | Determines whether an announcement should be shown (notification or dialog) — see `AnnouncementsStatus`. |
| `checkCredits` | Resolves the credits count. |
| `checkEditorType` | Resolves the current `Editor` (figma/figjam/dev/dev_vscode/buzz/penpot/sketch/framer). |
| `checkTrialStatus` | Resolves `TrialStatus` + `trialRemainingTime`. |
| `checkUserConsent` | Checks consent against `versions.userConsentVersion`. |
| `checkUserLicense` | Checks license status (triggers a `LicenseTrigger` if needed: `ACTIVATE`, `JUMP`, `CUSTOM_CHECKOUT`). |
| `checkUserPreferences` | Loads user preferences. |

## 4. Trial

`bridges/plans/enableTrial.ts` + `TrialEvent { date, trialTime }`. A trial has a duration (`Config.plan.trialTime`) and a 4-state status (`UNUSED → PENDING → EXPIRED|SUSPENDED`).

## 5. Consent

`ConsentConfiguration[]` (type `@unoff/ui`), versioned via `Config.versions.userConsentVersion`, stored in `stores/consent.ts`.

## See also

- [Glossary](../00-overview/glossary.md) — Platform / Editor / Plan / Consent terms
- [Preferences](../02-features/preferences.md) — language + deep-sync preferences
- [Modals](../02-features/modals.md) — pricing, license, trial, and consent-adjacent dialogs
- [Palette](palette.md) §6 — the publish/duplicate lifecycle, which relies on the identity fields above
