# BLE Device Console

A browser client that scans for, connects to, and controls a simulated BLE peripheral. Built with the Web Bluetooth API — single static HTML file, no build step, no dependencies.

## Profile

| Item | UUID | Properties |
|---|---|---|
| Service | `0000FF00-0000-1000-8000-00805F9B34FB` | — |
| Status characteristic | `0000FF01-0000-1000-8000-00805F9B34FB` | Read, Notify |
| Control characteristic | `0000FF02-0000-1000-8000-00805F9B34FB` | Write |

These are the UUIDs given in the assignment; both characteristic values are a single unsigned byte, since both ranges (0–100 and 0–255) fit in one byte.

## Why Web Bluetooth, and the catch

I picked Web Bluetooth because it let me get to real GATT communication fast, with the browser handling the scanning UI and permission flow, and because testing it is trivial — open the file, click connect, done. That speed is the point of a 3–5 hour exercise.

The catch, and it's a real one: **Safari and Firefox don't implement Web Bluetooth at all**, on any platform. Since the team's actual MVPs are native iOS and Android, this prototype could never ship as-is to an iOS user — there's no polyfill that gives a normal Safari tab access to CoreBluetooth. If this were heading to production rather than a take-home, the BLE logic here (the read/notify/write/disconnect sequencing) would need to be re-implemented against CoreBluetooth on iOS and the Android BLE APIs (or React Native / Flutter wrapping those) rather than the browser. Flagging this upfront rather than letting the demo imply otherwise.

## Simulating the peripheral

Using nRF Connect for Mobile (Nordic Semiconductor) on a spare phone:

1. Configure a GATT server with the service/characteristics above (Status = Read + Notify, Control = Write).
2. In the Advertiser tab, create an advertising packet that includes the **Service UUID** record for `0000FF00…`. This matters: the client filters its scan by service UUID, so the simulated device won't appear in the picker if the service UUID isn't in the advertisement.
3. Start advertising.

Android's nRF Connect is the more reliable choice for the GATT server / advertiser feature. iOS 16+ supports it too but has had occasional notification-delivery bugs in some versions.

## Running the client

`index.html` is the whole app.

- **Open it directly.** Double-click it to open in Chrome — `file://` origins are treated as secure contexts, so Web Bluetooth works without a server in most cases.
- **Or serve it locally** if the direct-open approach acts up: `python3 -m http.server 8000` from the folder, then visit `http://localhost:8000`.

Requirements: a Chromium-based browser (Chrome, Edge, Opera) on desktop or Android; Bluetooth turned on; and the page open in a real top-level tab, not embedded in another site's iframe (the Bluetooth permissions policy defaults to same-origin only).

## What's implemented

Core requirements: scans and connects by Service UUID, reads and displays Status on connect, subscribes to Status notifications and updates the UI live, writes a 0–255 value to Control from a slider, and detects disconnects with a visible connection-state indicator (the pill at the top — disconnected / connecting / connected / error).

Stretch goals, all implemented:

- **Auto-reconnect.** A toggle (on by default) controls whether an unexpected disconnect triggers automatic retries. Retries use exponential backoff (1s, 2s, 4s, 8s, 8s — capped) up to 5 attempts before giving up and surfacing an error state. A manual "Disconnect" click is tracked separately from an unexpected drop, so clicking Disconnect never triggers a reconnect loop against your own intent, and a "Cancel reconnect" button appears mid-retry if you want to bail out early.
- **Edge-case handling.** `navigator.bluetooth.getAvailability()` is checked on load to detect a missing Bluetooth adapter outright. Errors from `requestDevice()`, `connect()`, and GATT operations are classified by `DOMException` name and message (cancelled picker vs. no matching device vs. permission/security block vs. lost-in-range vs. wrong GATT profile on the target device) and shown as a specific, actionable log line rather than a generic failure.
- **Remember last device.** On load, the app calls `navigator.bluetooth.getDevices()` to ask the browser for devices it has already granted this origin permission to use. If one exists, a "Reconnect to last device" button appears that connects directly — no picker, no re-prompt.

## Hardening this for production

### Reconnection strategy

What's here: backoff with a cap, a max-attempt ceiling, and a clean separation between "I meant to disconnect" and "the connection dropped on me," which is the part that's easy to get wrong and ends up either spamming reconnects after a deliberate disconnect or silently giving up after a real one.

What I'd add for production: this only works while the tab is open and foregrounded — Web Bluetooth (and browsers generally) don't grant background execution, so there's no equivalent of an Android foreground service keeping a connection alive while the user is in another app. A real mobile client needs that: on Android, a bound/foreground service maintaining the GATT connection independent of activity lifecycle; on iOS, CoreBluetooth's background modes and `CBCentralManager` state restoration so the OS can hand the connection back to your app after backgrounding or even a relaunch. I'd also tune retry behavior per failure type rather than treating all disconnects the same — a connection lost because the device walked out of range should retry; one lost because the user turned off Bluetooth shouldn't burn through five attempts in eight seconds for no reason. And for a fleet of devices rather than one, I'd want jitter on the backoff so a power outage that drops twenty devices at once doesn't cause them all to hammer the gateway in lockstep.

### Error handling

What's here: errors are classified by name/message into a handful of buckets (cancelled, no device found, security/permission, GATT profile mismatch, lost connection) with a specific message for each, logged with a timestamp.

What I'd add for production: the categories here are inferred from `DOMException.message` text, which is not a stable contract — it's a best-effort heuristic, not something to build critical logic on. A production app needs structured, versioned error codes from whatever BLE layer it's using (native APIs give cleaner status codes than the browser does), telemetry on failure rates and types so regressions are visible without a user filing a ticket, and a clear user-facing distinction between "retry automatically" (transient, like out-of-range) and "stop and tell the user something" (permission denied, adapter off, wrong firmware) instead of leaving that distinction implicit in code comments.

### Multiple devices

What's here is deliberately single-device: the profile in this assignment is one peripheral, and the state (`device`, `statusChar`, `controlChar`, reconnect counters) lives in a handful of module-level variables, which is fine for one connection and wrong for more than one.

What I'd change: that state needs to move into a per-device session object — something like a `Map<deviceId, DeviceSession>` where each session owns its own GATT references, its own reconnect attempt counter and backoff timer, and its own notification listeners, so one device's retry storm or disconnect doesn't touch another's state. The UI would need to become a device list rather than a single status block. And there's a real hardware ceiling to plan around: BLE central radios on phones typically cap concurrent GATT connections somewhere around 4–7 depending on chipset, so "multiple devices" at any real scale needs either a gateway/hub model (one always-on device holding the BLE connections and relaying over Wi-Fi/cloud to the app) or an explicit, user-visible limit rather than letting the tenth connection attempt fail mysteriously.

## What I'd do with more time

I'd build a small scriptable virtual peripheral (a Python service using `bleak`'s peripheral mode, or a BlueZ D-Bus script) instead of relying on manually clicking through nRF Connect for every test run — that's the difference between testing being a five-minute manual ritual and something that runs in CI. I'd add a thin abstraction layer between the UI and the raw Web Bluetooth calls so the GATT logic could be unit-tested against a mock adapter rather than only ever exercised against real hardware. And given the team's actual stack, I'd want to spend real time on a native iOS (CoreBluetooth) spike specifically to find where the platform differences bite — connection state callbacks, background restoration, and characteristic write timing all behave differently enough from Web Bluetooth's promise-based API that I wouldn't want to assume this design ports over cleanly without checking.

## Known limitations

Web Bluetooth isn't supported in Firefox or Safari (a platform constraint, not a bug here). `navigator.bluetooth.getDevices()` has shipped at different stability levels across Chrome versions — this code feature-detects it and degrades gracefully (the "Reconnect to last device" button just doesn't appear) rather than assuming it's always available. Auto-reconnect only runs while the tab is open in the foreground, for the background-execution reasons described above.

