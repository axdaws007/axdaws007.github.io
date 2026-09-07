# axdaws007.github.io

This site exists for **one purpose**: to serve
`/.well-known/assetlinks.json`, the Android App Links digital asset link
for the CleaningApp mobile application.

## Why it exists

CleaningApp signs people in through Microsoft Entra External ID. The
sign-in result comes back to the app through a redirect URI.

A **custom URI scheme** redirect (`io.github.axdaws007.cleaningapp://…`)
is not exclusive on Android — any app on the device may register the
same scheme and intercept the authorisation code. Because the identity
provider cannot tell which app is about to receive the result, it shows
the person an interstitial asking them to vouch for the app:

> Are you trying to sign in to CleaningApp Mobile?
> Only continue if you downloaded the app from a store or website that
> you trust.

An **app-claimed HTTPS redirect** (an Android App Link) removes that
question, because the operating system verifies the claim
cryptographically instead of asking a person to. Android fetches the
file below over HTTPS, checks the signing-certificate fingerprint
against the installed app, and only then routes the URL to it.

This was established by measurement on 2026-09-07 rather than assumed:
the same authorisation request produced the interstitial with a custom
scheme redirect and **did not** produce it with an HTTPS redirect, with
the custom scheme run as a control in the same session.

## What is in here

| Path | What |
|---|---|
| `.well-known/assetlinks.json` | The digital asset link. Package name plus the SHA-256 fingerprints of every signing certificate allowed to claim these URLs |
| `cleaningapp/oauth2redirect/` | The redirect target. A person reaching it in a browser has no app installed to intercept it, so it explains itself rather than showing a blank page |

## The fingerprints, and the trap

`sha256_cert_fingerprints` is **per signing keystore**. The debug and
release builds of an Android app are signed with different keys and
therefore have different fingerprints.

**Both must be listed**, or the build that is missing silently stops
verifying — App Links fails open to the browser rather than erroring,
so the symptom is "sign-in started opening a web page again" with
nothing in any log.

Read a fingerprint with:

```
keytool -list -v -keystore <keystore> -alias <alias>
```

The debug keystore is at `~/.android/debug.keystore`, alias
`androiddebugkey`, store and key password `android`.

**Nothing in this file is secret.** Signing-certificate fingerprints are
public by design — they are how the platform verifies a claim, not how
it authenticates one.

## Verifying a change

Android's verifier is strict about the fetch. After editing:

```
curl -sSI https://axdaws007.github.io/.well-known/assetlinks.json
```

It must be `200`, served over HTTPS with a valid certificate,
`content-type: application/json`, and **with no redirect** on the way —
including `http` → `https`.

On a device with the app installed:

```
adb shell pm verify-app-links --re-verify io.github.axdaws007.cleaningapp
adb shell pm get-app-links io.github.axdaws007.cleaningapp
```
