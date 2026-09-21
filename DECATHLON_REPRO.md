# Reproduction: SFMCSdk.configure() called twice crashes the SDK

**Reported by:** Decathlon Android team
**Base commit:** `8c8df5f` (`master`, "MCET 260" — SDK `11.0.+`, matching our
production SDK version `11.0.1` almost exactly)
**Context:** production crashes seen in `com.decathlon.app`
(`javax.crypto.IllegalBlockSizeException` / `KeyStoreException`, and
`IllegalStateException: attempt to re-open an already-closed object` on the
SDK's internal SQLite storage) whenever `SFMCSdk.configure()` ends up being
invoked twice in quick succession during app cold start.

## What was changed

Exactly one file in this stock, unmodified `LearningApp` sample was
touched: `BaseLearningApplication.kt`. The only change is calling
`SFMCSdk.configure()` twice in a row instead of once, inside a
`repeat(2) { ... }` block clearly marked
`--- DECATHLON REPRO ---` / `--- END DECATHLON REPRO ---`. No other file,
no app-level business logic, no third-party code is involved.

See the diff on this branch (`repro/double-configure-crash`) relative to
`master`.

## How to reproduce

1. Fill in `gradle.properties` with a **real** MobilePush environment
   (`MC_APP_ID`, `MC_ACCESS_TOKEN`, `MC_SENDER_ID`, `MC_MID`,
   `MC_SERVER_URL`) — any valid environment reproduces this, it is not
   specific to Decathlon's.
2. `./gradlew :app:assembleBasicDebug`
3. **Fresh install required each time** (`adb uninstall <applicationId>`,
   then `adb install app-basic-debug.apk`) — the race only manifests on the
   very first `configure()` call after install, while the SDK's Keystore
   encryption key doesn't exist yet and both calls race to generate/read
   it. Subsequent launches without a fresh install won't reproduce it,
   since the key is by then cached.
4. Launch the app and check logcat for `KeyStoreException`,
   `IllegalBlockSizeException`, or a `NullPointerException:
   getEncryptionKey(...) must not be null`.

## Result observed by Decathlon

6/6 crashes across 6 independent fresh-install runs on two different
devices (one emulator, one physical Pixel 9), with two distinct concrete
exceptions surfacing from the same missing synchronization inside
`SFMCSdk.configure()`. This was first validated on SDK `9.0.3` (core
`SFMCSdk v1.0.5`); the repro logic on this branch has since been rebased
onto the latest `master` (SDK `11.0.+`, core `SFMCSdk v1.0.6`), the exact
version range used in Decathlon's production app, and **re-validated 5/5
on a freshly wiped emulator** (`emulator -wipe-data`) — same signature
(`KeyStoreException: Key not found` → `NullPointerException:
getEncryptionKey(...)`).

| Device / SDK | Runs | Crashes | Signature(s) observed |
|---|---|---|---|
| Emulator (Pixel 3a AVD), SDK 9.0.3 | 3 | 3/3 | `KeyStoreException: Key not found` → `NullPointerException: getEncryptionKey(...)` |
| Physical device (Pixel 9), SDK 9.0.3 | 3 | 3/3 | `IllegalBlockSizeException` (2/3) and `KeyStoreException: Key not found` (1/3) → `NullPointerException: getEncryptionKey(...)` |
| Emulator (Pixel 3a AVD, **freshly wiped**), SDK 11.0.+ (`master`) | 5 | 5/5 | `KeyStoreException: Key not found` → `NullPointerException: getEncryptionKey(...)` |

⚠️ **Testing note:** a plain `adb uninstall` is not always enough to reset
the Keystore-key race window on a device that has already run this repro
before — on a physical device we'd hammered with earlier test runs, we
briefly got 0/3 "crashes" after rebasing onto SDK 11.0.+, which looked like
the newer SDK might have fixed the race. Re-testing on a **freshly wiped**
emulator (`emulator -avd <name> -wipe-data`) immediately reproduced 5/5,
confirming the SDK version wasn't the reason it stopped crashing — stale
Keystore state on the device was. Always use a genuinely fresh device/AVD
(wiped, not just uninstalled) when attempting this reproduction.

Every frame in the resulting stack traces is inside
`com.salesforce.marketingcloud.sfmcsdk.*` and Android's own
`android.security.keystore.*` — there is no Decathlon code anywhere in
this call path.

Full write-up, including production stack traces and root-cause analysis
of why our app ends up calling `configure()` twice, is available from the
Decathlon team on request.
