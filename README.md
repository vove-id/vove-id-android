# VoveID Android SDK 1.10.0

## Install

Add JitPack to your dependency repositories:

```kotlin
repositories {
    google()
    mavenCentral()
    maven { url = uri("https://jitpack.io") }
}
```

Add the SDK to your app:

```kotlin
dependencies {
    implementation("com.github.vove-id:vove-id-android:1.10.0")
}
```

The SDK supports Android API 24 and later. Your backend supplies the verification
session token; initialize the SDK with your organization's public key before launch.

## Activity Result API

From 1.10.0, use the Android Activity Result API to deliver verification results to
the current registered activity or fragment, including a recreated caller.
Register the launcher unconditionally whenever the caller is created, in a stable
order relative to any other launchers:

```kotlin
import com.voveid.sdk.Vove
import com.voveid.sdk.VoveEnvironment
import com.voveid.sdk.activityresult.VerificationOutcome
import com.voveid.sdk.activityresult.VerificationRequest
import com.voveid.sdk.activityresult.VoveVerificationContract

private val verification = registerForActivityResult(VoveVerificationContract()) { result ->
    when (result.outcome) {
        VerificationOutcome.INTERRUPTED -> reconcileWithYourBackend(result.referenceId)
        else -> handleVerificationResult(result)
    }
}

// In response to an explicit user action, after obtaining a session from your backend:
Vove.initialize(this, VoveEnvironment.SANDBOX, organizationPublicKey) { ready ->
    if (ready) {
        runOnUiThread {
            verification.launch(
                VerificationRequest(
                    sessionToken = sessionToken,
                    requestId = savedRequestId,
                    referenceId = backendVerificationReference,
                )
            )
        }
    }
}
```

`reconcileWithYourBackend` and `handleVerificationResult` are your app's handlers.
Save the request ID, backend reference, and caller navigation state before launch
(for example in `SavedStateHandle`). Do not create a new verification or relaunch
automatically when Android restores the caller.

Results include the outcome, request/reference IDs, next step, interruption reason,
and an optional support action. Normal outcomes are `SUCCESS`, `FAILURE`, `PENDING`,
`IN_PROGRESS`, `CANCELLED`, and `MAX_ATTEMPTS_REACHED`.

## Interruption and backend reconciliation

After actual process death, restoring the existing Android task returns
`INTERRUPTED / PROCESS_RESTARTED` instead of restoring unsafe camera state.
Other interruption reasons cover missing SDK initialization, lost activity state,
and unavailable or malformed platform results. If Android returns no SDK result
Intent, correlation IDs can be null; use the IDs saved by your caller.

This release safely interrupts the flow; it does not resume mid-capture. Android
must restore the task and your app must register its launcher again to receive
the result. Force-stop or removal of the task cannot guarantee a callback.

Use backend webhooks and fetch the current backend verification status on return
before deciding whether to retry. An interruption does not mark the verification
failed or canceled on the backend. A local next-step result is not independent
proof of server acceptance.

## Existing callback integrations

The existing `Vove.start(...)` overloads, options, payload, callback interfaces, and
`VerificationResult` enum remain available. Normal legacy completion invokes its
callback once; legacy support actions invoke only the support callback.

In-memory legacy callbacks cannot be restored after process death or rebound to
a recreated caller. Adopt `VoveVerificationContract` for lifecycle-aware delivery
and explicit interruption reasons. Existing callbacks use `CANCELLED` if an
interruption occurs while their callback is still available.
