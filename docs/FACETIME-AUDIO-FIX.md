# FaceTime audio-process ownership fix

## Scope

This change makes the existing per-app mixer include a live audio process only
when there is evidence that it belongs to exactly one running regular app. It
does not add a FaceTime process-name exception, a virtual device, a driver, a
service, microphone capture, or any change to the system output volume.

## What the source proved

`AppVolumeMixer.readSnapshot` grouped an audio object only after
`ResponsibleProcess.regularAppOwner` found a regular app. That lookup used the
responsibility API and a bounded BSD parent walk. A helper whose responsibility
and parent chain both end outside a regular app was therefore omitted before
its Core Audio bundle identifier was read. The existing bundle-identifier read
could only fill in a persistence key after an owner had already been found.

The existing routing path is otherwise deliberate: when an app differs from
100% or uses a non-default output, `TapGainEngine` creates a muted process tap
for that row's `audioObjects` and replays the scaled samples to the requested
output. At 100% on the default output it creates no tap and leaves the app on
the normal system path.

## Change

`MixerRoutingSupport.regularAppOwnerPid` now resolves ownership in this order:

1. The responsibility API and BSD parent chain, exactly as before.
2. If that fails, the public Core Audio `kAudioProcessPropertyBundleID` for the
   live audio object, but only if it identifies exactly one currently-running
   regular app.

`AppVolumeMixer` supplies that HAL property while it builds the row. A matching
object is therefore included in the existing row's process tap; an unowned or
ambiguous helper is still not tapped. Ownership is recomputed on every mixer
snapshot, so no helper PID or `AudioObjectID` becomes a durable identity.

This is a narrow source-discovery repair. It does not prove that a particular
FaceTime call helper reports FaceTime's bundle identifier on every macOS build.
If the live helper reports only its own shared-service bundle and has no
responsibility or parent-chain link to FaceTime, public evidence is insufficient
to claim it as FaceTime, and the mixer correctly leaves it untouched.

## Files changed

- `Sources/Vorssaint/Services/Audio/MixerRoutingSupport.swift` — pure,
  testable owner-resolution rule with an ambiguity-safe bundle fallback.
- `Sources/Vorssaint/Services/ResponsibleProcess.swift` — supplies current
  regular-app candidates to that rule.
- `Sources/Vorssaint/Services/Audio/AppVolumeMixer.swift` — reads each live
  Core Audio process object's bundle identifier before ownership grouping.
- `Tests/MetricsTests.swift` — covers exact helper ownership, ambiguity,
  snapshot/PID refresh, zero gain, and unity gain.

## Verification performed

Environment inspected on 2026-09-16:

- macOS 26.6.2 (25G83), Apple Silicon (`arm64`).
- Apple Swift 6.2.3; macOS SDK 26.2.
- `system_profiler SPAudioDataType` exposed no output device to this development
  session. Dell model, transport, Core Audio device ID, sample rate, channel
  layout, and hardware-volume capabilities are therefore unknown.

The initial sandboxed `./build.sh --test` could not read the CPU-count sysctl
and failed before compiling (`invalid value '' in '-j'`). This was an execution
environment restriction, not a project test result. The same command in the
normal macOS environment passed after the change:

```
./build.sh --test
# TESTS OK (50595 checks)
# PREFERENCE CLEANUP TESTS OK
```

The full developer build also passed:

```
./build.sh --dev
# Bundle ready: build/stage.noindex/Vorssaint (Developer).app
```

The bundle passed `codesign --verify --deep --strict`, and its actual
executable completed `--selftest` with `SELFTEST OK`:

```
build/stage.noindex/Vorssaint (Developer).app
build/stage.noindex/Vorssaint (Developer).app/Contents/MacOS/VorssaintDeveloper
```

The new tests establish that an unparented live audio object joins one exact
regular-app bundle owner, never an ambiguous or unowned helper, and that a new
snapshot uses the current owner PID. Existing mixer tests also cover engine
build de-duplication, stale engine recovery, output-device changes, cleanup,
channel mapping, and non-audio input buffers. The added render checks confirm
that gain 0 writes silence and gain 1 preserves samples before any boost
limiting.

## Hardware acceptance still required

No controlled FaceTime call, Dell output, or microphone path was available to
this task. Scenarios A–J in the request are **not passed by this document**.
In particular, the following remain unverified on real hardware:

- FaceTime's incoming call audio is represented by an object that passes the
  new ownership rule.
- 0% silences only that incoming audio while 100% restores its normal level.
- Dell output level, hardware/DDC settings, the system master volume, other
  app gains, microphone transmission, echo cancellation, and call quality stay
  unchanged.
- Call restart, output change, Dell reconnect, sleep/wake, and mixer shutdown
  leave no stale or double tap.

## Safe manual acceptance test

1. Quit other mixers. Leave the Dell output selection and physical volume
   unchanged, then record a repeatable reference level with the mixer off.
2. Launch the developer bundle above. Grant **System Audio Recording** only to
   that developer app if macOS asks; do not reset TCC permissions globally.
3. With the other participant's consent, start a FaceTime call and play a
   steady, separate browser or music reference. Set both rows to 100% first.
4. Lower the FaceTime row to 50%, then 0%, while holding the other row steady.
   At 0%, the remote voice must be inaudible; the reference audio must not
   change. Restore FaceTime to 100% and confirm the initial level returns.
5. While FaceTime is at 0%, confirm the remote participant can still hear the
   local microphone. Then end and restart the call, change output if supported,
   and finally quit the mixer to confirm ordinary audio returns.

Treat a slider movement, a successful API return, or silence caused by a call
pause as insufficient evidence. If 0% still leaves the remote voice audible,
capture only non-content diagnostic metadata (the live audio-object PID and
bundle identifier, responsibility/parent result, and tap render count); do not
record call audio or participant information.

## Rollback

The change is confined to the four files listed above. Revert its local commit
with `git revert <commit>` after it is committed, then rebuild with
`./build.sh --dev`. This does not alter the official app, user preferences, or
TCC state.
