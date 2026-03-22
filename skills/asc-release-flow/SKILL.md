---
name: asc-release-flow
description: Determine whether an app is ready to submit, then drive the App Store release flow with asc, including first-time submission fixes for availability, subscriptions, and App Privacy.
---

# Release flow (readiness-first)

Use this skill when the real question is "Can my app be ready to submit?" and then guide the user through the shortest path to a clean App Store submission, especially for first-time releases.

## Preconditions
- Ensure credentials are set (`asc auth login` or `ASC_*` env vars).
- Resolve app ID, version string, and build ID up front.
- Have a metadata directory ready if you plan to use `asc release run`.
- If you use experimental web-session commands, use a user-owned Apple Account session and treat those commands as optional escape hatches, not the default path.

## How to answer
When using this skill, answer readiness questions in this order:
1. Is the app ready right now, or not yet?
2. What are the blocking issues?
3. Which blockers are API-fixable vs web-session-fixable?
4. What exact command should run next?

Group blockers like this:
- API-fixable: build validity, metadata, screenshots, review details, content rights, encryption, version/build attachment.
- Web-session-fixable: initial app availability bootstrap, first-review subscription attachment, App Privacy publish state.
- Manual fallback: only when the user does not want to use experimental web-session commands or Apple UI behavior has changed.

## Canonical path

### 1. Fast readiness check
Run this first when the user wants the quickest answer to "can I submit now?":

```bash
asc submit preflight --app "APP_ID" --version "1.2.3" --platform IOS
```

This is the fastest high-signal readiness check and prints fix guidance without mutating anything.

### 2. Full-pipeline dry run
Run this when the user wants one command that approximates the whole release path:

```bash
asc release run \
  --app "APP_ID" \
  --version "1.2.3" \
  --build "BUILD_ID" \
  --metadata-dir "./metadata/version/1.2.3" \
  --dry-run \
  --output table
```

This is the best single-command rehearsal for:
1. ensuring or creating the version
2. applying metadata and localizations
3. attaching the build
4. running readiness checks
5. confirming the submission path is coherent

Add `--strict-validate` when you want warnings treated as blockers.

### 3. Actual submit
When the dry run looks clean:

```bash
asc release run \
  --app "APP_ID" \
  --version "1.2.3" \
  --build "BUILD_ID" \
  --metadata-dir "./metadata/version/1.2.3" \
  --confirm
```

## First-time submission blockers

### 1. Initial app availability does not exist yet
Symptoms:
- `asc pricing availability get --app "APP_ID"` reports no availability
- `asc pricing availability set ...` fails because it only updates existing availability

Check:

```bash
asc pricing availability get --app "APP_ID"
```

Bootstrap the first availability record with the web-session flow:

```bash
asc web apps availability create \
  --app "APP_ID" \
  --territory "USA,GBR" \
  --available-in-new-territories true
```

After bootstrap, use the normal API command for ongoing updates:

```bash
asc pricing availability set \
  --app "APP_ID" \
  --territory "USA,GBR" \
  --available true \
  --available-in-new-territories true
```

### 2. Subscriptions are READY_TO_SUBMIT but not attached to first review
For apps with subscriptions, check readiness explicitly:

```bash
asc validate subscriptions --app "APP_ID" --output table
```

List current first-review subscription state:

```bash
asc web review subscriptions list --app "APP_ID"
```

If the app is going through its first review and the group needs attaching:

```bash
asc web review subscriptions attach-group \
  --app "APP_ID" \
  --group-id "GROUP_ID" \
  --confirm
```

For one subscription instead of a whole group:

```bash
asc web review subscriptions attach \
  --app "APP_ID" \
  --subscription-id "SUB_ID" \
  --confirm
```

For later reviews, use the normal submission path:

```bash
asc subscriptions review submit --subscription-id "SUB_ID" --confirm
```

### 3. App Privacy is still unpublished
The public API can warn about App Privacy readiness but cannot fully verify publish state.

If `asc submit preflight`, `asc validate`, or `asc release run` surfaces an App Privacy advisory, reconcile it with:

```bash
asc web privacy pull --app "APP_ID" --out "./privacy.json"
asc web privacy plan --app "APP_ID" --file "./privacy.json"
asc web privacy apply --app "APP_ID" --file "./privacy.json"
asc web privacy publish --app "APP_ID" --confirm
```

If the user does not want the experimental web-session flow, confirm App Privacy manually in App Store Connect:

```text
https://appstoreconnect.apple.com/apps/APP_ID/appPrivacy
```

### 4. Review details are incomplete
Check whether the version already has review details:

```bash
asc review details-for-version --version-id "VERSION_ID"
```

If needed, create or update them:

```bash
asc review details-create \
  --version-id "VERSION_ID" \
  --contact-first-name "Dev" \
  --contact-last-name "Support" \
  --contact-email "dev@example.com" \
  --contact-phone "+1 555 0100" \
  --notes "Explain the reviewer access path here."
```

```bash
asc review details-update \
  --id "DETAIL_ID" \
  --notes "Updated reviewer instructions."
```

Only set `--demo-account-required=true` when App Review truly needs demo credentials.

## Practical readiness checklist
An app is effectively ready to submit when:
- `asc submit preflight --app "APP_ID" --version "VERSION"` reports no blocking issues
- `asc release run ... --dry-run` produces the expected plan
- the build is `VALID` and attached to the target version
- metadata, screenshots, and localizations are complete
- content rights and encryption requirements are resolved
- review details are present
- app availability exists
- subscriptions, if any, are attached for first review or already submitted through the supported review path
- any App Privacy advisory has been resolved through `asc web privacy ...` or manual confirmation

## Lower-level fallback
Use the lower-level flow only when the user needs explicit control over each step:

```bash
asc versions attach-build --version-id "VERSION_ID" --build "BUILD_ID"
asc submit preflight --app "APP_ID" --version "1.2.3" --platform IOS
asc submit create --app "APP_ID" --version "1.2.3" --build "BUILD_ID" --confirm
asc status --app "APP_ID"
```

## Platform notes
- Use `--platform MAC_OS`, `TV_OS`, or `VISION_OS` as needed.
- For macOS, upload the `.pkg` separately, then use the same readiness and submission flow.
- `asc publish testflight` is still the fastest TestFlight shortcut, but for App Store readiness prefer `asc submit preflight` and `asc release run`.

## Notes
- `asc release run --dry-run` is the closest thing to a one-command answer for "will this release flow work?"
- `asc submit preflight` is the fastest first pass.
- Web-session commands are experimental and should be presented as optional escape hatches when the public API cannot complete the first-time flow.
- If the user asks "why did submission fail?" map the failure back into the three buckets above: API-fixable, web-session-fixable, or manual fallback.
