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

## 2. Plan, trial, credits

```
PlanStatus   = 'UNPAID' | 'PAID' | 'NOT_SUPPORTED'
TrialStatus  = 'UNUSED' | 'PENDING' | 'EXPIRED' | 'SUSPENDED'
```

`BaseProps` (context injected into the UI) carries `planStatus`, `trialStatus`, `trialRemainingTime`, `creditsCount`, `creditsRenewalDate`.

`Config.plan` defines the global parameters: `isProEnabled`, `isTrialEnabled`, `isCreditsEnabled`, `trialTime`, `creditsLimit`, `creditsRenewalPeriodDays`, plus store IDs (`storeProWeekId`, `storeProMonthId`, `storeProYearId`, `storeProLifetimeId`).

**⚠️ Confirmed by the author (2026-07-27): the credits system is currently inactive in production** (`isCreditsEnabled` disabled at the product level), even though all the credit-gating code remains in place in `ui-ui-color-palette` (checks, optimistic debits, `Config.fees`). It could become active again in the future — do not treat credit-related code paths as dead code to remove, and do not conclude from a lack of test coverage/references to credits in a module that they're out of scope for future development.

### Actions consuming credits (`Config.fees`)

| Action | `fees` field |
| --- | --- |
| Coolors import | `coolorsImport` |
| Colour Lovers import | `colourLoversImport` |
| Realtime Colors import | `realtimeColorsImport` |
| Extraction from image | `imageColorsExtract` |
| Harmony creation | `harmonyCreate` |
| AI generation | `aiColorsGenerate` |
| Palette creation | `paletteCreate` |
| Palette generation | `paletteGenerate` |
| Palette generation with properties | `paletteWithPropsGenerate` |
| Sheet generation | `sheetGenerate` |
| Palette updates | `paletteUpdates` |
| Local styles sync | `localStylesSync` |
| Local variables sync | `localVariablesSync` |
| Local tokens sync | `localTokensSync` |

Any new product-side action must explicitly decide whether it belongs in this table (consumes credits) or not — to settle in the corresponding feature spec, "Impact" section.

## 3. Guardrails before an action (`src/bridges/checks/`)

| Check | Likely role |
| --- | --- |
| `checkAnnouncementsStatus` | Determines whether an announcement should be shown (notification or dialog) — see `AnnouncementsStatus`. |
| `checkCredits` | Verifies the user has enough credits before a billed action. |
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
- [Preferences](../02-features/preferences.md) — language + deep-sync preferences, plan-gated the same way as here
- [Modals](../02-features/modals.md) — pricing, license, trial, and consent-adjacent dialogs
- [Palette](palette.md) §6 — credits interact with the publish/duplicate lifecycle documented there
