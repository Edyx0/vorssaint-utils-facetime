# FaceTime conference-audio handling

## Scope

This change makes the existing per-app mixer handle two distinct, evidence-led
cases without changing system output volume, monitor volume, microphone input,
or any app's saved preferences:

1. A live Core Audio process whose responsibility, parent chain, or exact
   bundle identifier identifies one running regular app continues to join that
   app's row.
2. A live, unowned `com.apple.avconferenced` process is shown as a temporary
   **Conference audio** source. It is deliberately not called FaceTime and it
   cannot be assigned a separate output.

There is no virtual device, driver, process-name-to-FaceTime attribution,
microphone capture, or global volume adjustment.

## What the live machine proved

On the inspected macOS 26.6.2 Apple-silicon machine, while FaceTime was open,
the Core Audio process list showed both:

| Process | Core Audio bundle | Responsibility / parent | Running output |
| --- | --- | --- | --- |
| FaceTime | `com.apple.FaceTime` | itself / `launchd` | no |
| `avconferenced` | `com.apple.avconferenced` | itself / `launchd` | yes (at capture time) |

The active source therefore had no public responsibility or parent-chain
evidence tying it to FaceTime. A bundle fallback must not fabricate that link:
`avconferenced` is a shared system conference-media daemon and may be used by
other features. The mixer exposes only the exact live source, with a
session-only identity, so the user can control its current output without a
false per-app claim or durable preference.

When the developer app was launched for the Dell test, the daemon was no
longer reporting `isRunningOutput`; consequently its temporary row correctly
did not appear. That prevents a stale Conference audio control after a call or
audio activity has stopped.

## Implementation

`MixerRoutingSupport.regularAppOwnerPid` retains the conservative resolution
order:

1. Responsibility API and bounded BSD parent-chain owner.
2. `kAudioProcessPropertyBundleID`, only when it identifies exactly one
   currently-running regular app.

`AppVolumeMixer.readSnapshot` reads the Core Audio bundle identifier before
ownership grouping. If normal ownership still fails, it calls
`transientSystemAudioSource`. That helper accepts only a currently-outputting
`com.apple.avconferenced` object and supplies a session-only
`system:conference-audio` row named **Conference audio**. Its gain is handled
by the existing process tap and replay path, but its output picker is hidden
and its volume is never written to `UserDefaults`.

At 100% on the current default output, this row has no tap. Lowering it creates
the existing muted process tap for precisely its current Core Audio object and
replays only that source at the chosen gain. Returning to 100% tears the tap
down. Input and microphone streams are not part of the source list.

## Files changed

- `Sources/Vorssaint/Services/Audio/MixerRoutingSupport.swift` — ambiguity-safe
  regular-app resolution plus a narrow, live shared-conference-source rule.
- `Sources/Vorssaint/Services/ResponsibleProcess.swift` — feeds active
  regular-app candidates into ownership resolution.
- `Sources/Vorssaint/Services/Audio/AppVolumeMixer.swift` — discovers the live
  source, makes its control session-only, and rejects output routing for it.
- `Sources/Vorssaint/UI/MenuPanel/MixerSection.swift` — hides the output picker
  for shared system sources.
- `Tests/MetricsTests.swift` — covers exact ownership, ambiguity, refresh,
  active/idle shared-source discovery, zero gain, and unity gain.

## Verification performed

The environment was inspected on 2026-09-16:

- macOS 26.6.2 (25G83), Apple Silicon (`arm64`); Swift 6.2.3; macOS SDK 26.2.
- Dell S2725QS connected through HDMI, stereo (2-channel), 44.1 kHz.
- The system initially used MacBook Pro Speakers. The test temporarily made
  Dell the default app output, then restored both default app and system output
  to MacBook Pro Speakers. The Sound panel was restored to **Selected Sound
  Output Device** for sound effects.
- No Core Audio volume, mute, balance, DDC, or monitor-hardware control was
  written. The MacBook output-volume UI remained at 56%, unmuted, with centred
  balance before and after restoration; Dell's physical volume was untouched.

Automated verification passed after the source change:

```text
./build.sh --test
TESTS OK (50598 checks)
PREFERENCE CLEANUP TESTS OK

./build.sh --dev
Bundle ready: build/stage.noindex/Vorssaint (Developer).app
```

The developer bundle launched successfully, completed onboarding, and showed
the mixer with Dell as the temporary default output. It could not show the
Conference audio row because the live source had become idle; that is the
expected inactive-path result.

## Hardware acceptance still required

A controlled FaceTime call with continuous remote audio was not available at
the instant the developer build ran. Therefore these are **not yet passed**:

- Lowering active Conference audio to 0% silences the remote conference source
  while other app audio stays unchanged.
- Restoring it to 100% restores its prior level without a duplicate or stale
  tap.
- Dell output level and call quality remain identical throughout that active
  call path.
- Call restart, output change, Dell reconnect, sleep/wake, and mixer shutdown
  cleanly refresh the temporary row.

For the remaining live check, start or resume a FaceTime call with a steady
remote voice, open the developer mixer's **Conference audio** row, and move
only that slider through 100% → 50% → 0% → 100%. Confirm that a separate
browser/music reference does not change and the remote party still hears the
local microphone. Do not record call audio; if the row is missing, collect
only PID, bundle identifier, responsibility/parent result, and tap-render
count.

## Rollback

The behavior is confined to the files listed above. Revert the local commits,
rebuild with `./build.sh --dev`, and no system volume, monitor setting, TCC
state, or user preference needs restoration.
