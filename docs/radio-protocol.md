# MeshPigeon Radio Protocol v2

The public, versioned contract between the MeshPigeon app and a MeshPigeon radio.
The radio is **dumb on purpose**: it receives packets into the largest memory
it can hold, stamps each with uptime, keys up on demand, and persists its
radio settings. It speaks **no mesh protocol** and holds **no keys**.

## 0. The spec is the `.proto` files

```
protobufs/meshpigeon/envelope.proto   envelopes, Error, Ping/Pong
protobufs/meshpigeon/device.proto     device info, settings, auth, status
protobufs/meshpigeon/radio.proto      radio tuning, packet store operations
```

Those three files **are** the wire contract: field numbers, oneof variants,
enum values and the per-field size limits (the `*.options` files that cap
every string/bytes field) are normative. This document is their prose
companion: framing mechanics, the semantics behind the fields, and the
evolution policy. Where the two disagree, the `.proto` files win.

`DeviceInfo.spec_version` is **2** for this document. A client must refuse a
radio whose `spec_version` it does not implement; the firmware never branches
on the client's version — it only reports its own.

- The wire payload of every frame is a serialized protobuf envelope
  (`ClientToRadio` or `RadioToClient`), identical on every transport.
- The firmware never parses packet payloads — raw bytes are all it keeps.
- The generated C sources are committed under
  `lib/meshpigeon-core/src/generated/` and the nanopb runtime is vendored in
  `lib/meshpigeon-core/src/nanopb/`, so a build needs nothing but PlatformIO.
  Regenerate with `scripts/regen-protos.sh` (CI fails on stale output).

## 1. Transports

| Transport | Bearer | Notes |
|---|---|---|
| USB CDC | virtual COM / serial | 115200 8N1 (line coding ignored); the primary bench/debug path. Carries **frames only** — a client must not expect a console on it, and a lost response is possible (see §2, "a lost ack") |
| BLE | Nordic UART Service (`6E400001-B5A3-F393-E0A9-E50E24DCCA9E`) | write-to-device char `6E400002-…`, notify-from-device char `6E400003-…`; multiple centrals on ESP32, single central on nRF52 |
| Wi-Fi TCP | raw TCP server, default port **5000** | multi-client (≤ 4), ESP32 boards only (§7); mDNS `_meshpigeon._tcp` |

BLE notify chunks frames at the negotiated MTU (≤ 20 bytes before an MTU exchange, so clients that never do one work). A frame is notified to the one central it is for — never to the others — and only once that central has enabled notifications.

**A connection is a connection, on every transport.** Auth state lives on
the connection, not on the transport: each BLE central gets its own, exactly
like each TCP socket does. A transport that shared one sink across several
links would hand a second central the session the first one authenticated,
which on the one transport that admits strangers without a password is the
whole attack. A push is one serialization however many are listening, and
each central is then sent its own copy.

**A lost ack is possible, and it is not always a retry.** Two operations on
the path can consume a response that was already produced: a frame dropped for
bad CRC or a bad COBS body, and a stray byte stream on the port that is not a
frame at all. §2 describes the second. The practical rule for a client is:

- **Idempotent operations** (`Get*`, `Ping`, `FetchPackets`, `PurgeStore`,
  `SendPacket`, `SetDeviceSettings`, `SetRadioSettings`) may simply be retried.
  `SendPacket` is idempotent in the sense that matters here — a lost ack leaves
  the *first* transmission on the air and the app's outbox still holds that
  `seq`, so a retry keyed the packet up a second time, which is the app's call
  to make, not a protocol error.
- **`Reboot`, `Bootloader` and `FactoryReset` must not be blind-retried.** A
  lost ack there means the node has already restarted, and re-sending restarts
  it again. Poll `DeviceInfo.boot_count` (or simply reconnect and re-read the
  state) to find out what happened.

## 2. Framing

```
wire frame   :=  COBS( envelope ‖ crc16 )  0x00
```

- **COBS**: consistent overhead byte stuffing; the 0x00 byte terminates each
  frame. Frames may be split across USB packets / BLE writes / TCP segments.
- **Nothing else may appear on these ports.** A client that finds a body which
  does not decode is looking at something that is not a frame, and it must not
  assume the bytes up to the next delimiter belonged to a frame it can trust.
  This is not hypothetical: the ESP-IDF Wi-Fi driver writes its own log text to
  the USB CDC port, and one such line silently consumes the frame that follows
  it.
- **CRC-16/CCITT-FALSE** (poly 0x1021, init 0xFFFF) over the serialized
  envelope, appended little-endian *before* the COBS pass.
- A frame with a bad CRC or a malformed COBS body is dropped silently; the
  next frame in the stream decodes normally.
- **The receiver hands the decoded envelope (CRC verified and stripped) to the
  command processor.** A maximum-size frame payload is **512 bytes**; a raw
  on-air packet is ≤ **255 bytes** (the Semtech SX12xx silicon cap). The 512
  is an exact boundary, not headroom: the largest legal envelope — a `Pong`
  echoing a 500-byte `Ping` with a 5-byte varint id — is exactly 512 bytes,
  and response encoding uses the same cap. A future change that grows an
  envelope past it makes the answer vanish with no error the client can see,
  so `test_max_size_pong_still_fits_the_frame` pins the edge.

Since v2 there is no `[cmd][nonce][status]` frame header. Correlation, the
operation and the error code all live *inside* the envelope:

- `ClientToRadio.id` — a client-chosen non-zero correlation id. Every response
  echoes it. An envelope that decodes but carries id 0 (or fails to decode at
  all) is dropped without a reply: there is no id to correlate an error to.
  "Fails to decode" includes any string or bytes field over its `*.options`
  cap (§8.1): the caps are the enforcement point, not a soft limit.
- Async pushes use `id = 0`.

## 3. Operations

The `body` oneofs in `envelope.proto` *are* the operation list. Each operation
has its own message, even the empty ones, so requests can gain fields later
without breaking older firmware (which skips what it does not know).

| Request | Response | Notes |
|---|---|---|
| `Ping` | `Pong` | echoes the payload bytes |
| `GetDeviceInfo` | `DeviceInfo` | identity, spec version, capabilities, store stats, health (§4) |
| `GetRadioSettings` | `RadioSettings` | current tuning |
| `SetRadioSettings` | `RadioSettings` | applies, bumps `config_epoch`, persists (§6) |
| `SendPacket` | `PacketAccepted` | then an async `TxResult` (§5) |
| `FetchPackets` | a `PacketEntry` per packet, then `FetchEnd` | stream, every frame echoes the request id |
| `PurgeStore` | `Ok` | history only; settings untouched |
| `GetDeviceSettings` | `DeviceSettings` | requires auth (§8) |
| `SetDeviceSettings` | `DeviceSettings` | full post-write read model (§8) |
| `GetStatus` | `Status` | **no auth** — this is how a BLE client learns what Wi-Fi is doing (§9) |
| `Auth` | `Ok` | unlocks *this* connection (§8) |
| `Reboot` | `Ok`, then restart | requires auth |
| `FactoryReset` | `Ok`, then wipe + restart | requires auth (§8) |
| `Bootloader` | `Ok`, then restart | requires auth. A plain restart today: real ROM/DFU entry is board work still to do (§13) |

Async pushes (`id = 0`), broadcast to every connected client:

| Message | Sent when |
|---|---|
| `PacketEntry` | a packet was received on the air (live push, same shape as a fetch entry) — **only to authorized connections**, since `FetchPackets` is gated and the payload is identical |
| `TxResult` | a `SendPacket` transmission finished |
| `RadioSettings` | another client re-tuned the radio (everyone except the issuer — **and only authorized connections**, since `GetRadioSettings` is gated) |
| `DeviceSettings` | another client changed device settings (everyone except the issuer — **and only connections that are authorized to see it**, since the read model carries the Wi-Fi passphrase) |
| `Status` | the Wi-Fi state changed (§9) |

The three gated pushes are gated for the same reason: an unauthenticated
socket is exactly as able to *listen* as it is to ask. A push is not an
operation, so nothing would stop it from carrying the same material the
request path refuses — and an async frame carrying the Wi-Fi passphrase, the
raw bytes of every packet the node hears, or the node's own tuning would
make the gate on the request meaningless. Each is filtered by the *same*
`is_authorized()` question its request path uses, so the two cannot drift
apart. All three still reach every connection while the device holds the
shipped default PIN, which is the open, unconfigured case (§8.2).

## 4. Errors

`Error.code` is normative in `envelope.proto`; `Error.message` is a short
human-readable hint, explicitly not a contract.

| Code | Meaning |
|---|---|
| `ERROR_CODE_BAD_COMMAND` | unknown operation (radio predates the oneof variant, or a bug) |
| `ERROR_CODE_BAD_PAYLOAD` | wrong size/format/validation — rejected atomically, nothing applied |
| `ERROR_CODE_BUSY` | TX in flight (a `SendPacket` or a re-tune), the first-owner lock is active (§6), or the packet store could not take the packet (§10) |
| `ERROR_CODE_TX_FAILED` | the radio refused a tuning (`SetRadioSettings`); a refused *transmission* is not an error at all — the send is accepted and reports `TxResult(success=false)` |
| `ERROR_CODE_NO_RADIO` | the radio failed to come up, or would not accept the persisted tuning at boot — there is no air to use |
| `ERROR_CODE_NOT_SUPPORTED` | feature absent on this board (capability-gated, §8) |
| `ERROR_CODE_AUTH_REQUIRED` | a PIN is set and this connection has not authenticated (§8) |

## 5. `DeviceInfo`

| Field | Meaning |
|---|---|
| `spec_version` | 2 |
| `fw_version`, `board_name` | strings ("XIAO WIO", "HELTEC V3", "T114", "T1000-E", "SIM") |
| `capabilities` | repeated enum: `WIFI_STA`, `BATTERY`, `BLE`, `USB_CDC` — compile-time facts, never probed |
| `uptime_ms` | monotonic 64-bit uptime (§5.1) |
| `boot_count` | real boots, persisted; informational only |
| `store` | `count`, `capacity_bytes`, `dropped`, `oldest_seq` |
| `radio_config_epoch` | bumped on every accepted `SetRadioSettings` |
| `battery_mv` | 0xFFFF = unknown (device status, not mesh telemetry) |
| `radio_ok` | false when the radio failed to come up at boot or would not accept the persisted tuning; `SendPacket` then answers `ERROR_CODE_NO_RADIO`. A re-tune the radio *refuses* does **not** clear it — the previous tuning stays in force |
| `auth_required` | true when a PIN is set and *this* connection is not authenticated |
| `noise_floor_dbm` | estimated receiver noise floor in dBm; **0 = unknown** (nothing received since boot) |

`noise_floor_dbm` is the **RSSI − SNR of the most recent received packet** —
the noise the receiver saw around the last signal it decoded. It is a reading,
not a mesh statistic, and it only changes when a packet arrives (there is no
periodic hardware noise probe in the port, so silence reports the last
estimate, or 0 before the first packet). Clients pair it with the per-packet
`rssi`/`snr` they already receive.

Client counts are **not** here: they change on every connect, so they live in
`Status` (§9), which is the snapshot clients poll.

### 5.1 Time

The radio keeps a monotonic **64-bit** `uptime_ms`: the 32-bit `millis()`
counter plus a RAM rollover count. It has no RTC and no wall clock — **time
correlation is the app's job**: on connect the app anchors radio uptime to app
time via `GetDeviceInfo` and re-anchors periodically. `PacketEntry.uptime_ms`
comes from the same clock, so every timestamp is plain arithmetic on the
client side — no rollover reasoning anywhere.

The rollover counter lives in RAM only (no flash writes for time, ever), so a
reboot legitimately restarts the clock and empties the store. Clients
re-anchor; that is the whole protocol for reboot detection.

## 6. Radio settings

`RadioSettings` speaks plain engineering units: `freq_hz` and `bandwidth_hz`
in Hz, `sf` 5..12, `cr` 5..8 (meaning 4/5..4/8), `power_dbm`. The v1 region
preset is **gone** — the app owns channels and legality; the firmware just
tunes what it is told.

- `SetRadioSettings` **persists immediately** (NVS / internal FS) and applies.
  A radio that reboots alone resumes listening with these settings. First
  boot ships a safe default and does not persist until an app tunes it.
- A retune attempted **while a transmission is keying up** answers
  `ERROR_CODE_BUSY`, like a second `SendPacket` does. It has to: the tuning
  sequence puts the radio in standby, which aborts the transmission on the
  silicon, so honouring the retune would mean the in-flight `SendPacket`
  never completes and never reports — leaving the node unable to transmit
  anything at all until it was power-cycled.
- The firmware **ignores the request's `config_epoch`** and bumps its own by 1
  on every accepted change; all *authorized* connected clients except the
  issuer get the async `RadioSettings` push, then the issuer gets the
  post-bump settings as its response. The push is filtered like every other
  gated material (§3) — `GetRadioSettings` is auth-gated, so a client that
  may not ask for the tuning has no business being told it.
- Out-of-range values (zero frequency/bandwidth, a bandwidth that is not a
  whole 10 Hz step, SF or CR out of range, TX power above **22 dBm**) are
  rejected with `ERROR_CODE_BAD_PAYLOAD`; a radio that refuses the applied
  settings answers `ERROR_CODE_TX_FAILED` and keeps the previous tuning,
  persisted and in force — so the device stays fully usable, and only that
  request fails. `bandwidth_hz` is checked *before* it is converted to the
  internal 0.01 kHz unit, so an out-of-range value can never wrap into a
  valid-looking one. `power_dbm` is bounded for the same reason: the value
  reaches the radio as an `int8_t`, so an unbounded `uint32` would not just
  be out of range — 200 would arrive as **−56 dBm** and the board would key
  up quietly at the wrong power instead of failing. Frequency is only
  checked for non-zero: channel legality is the app's call (§6).
- **Persistence is best-effort.** A flash write that fails is not reported
  (there is no error code for it, and NVS cannot report one reliably); the
  setting is applied in RAM and the response reflects that. The radio tuning
  is the only thing a device is useless without, and it is written only when
  an app asks for it.

### 6.1 First-owner lock

During the **first 5 minutes after boot**, only the *first*
`SetRadioSettings` is honored; further attempts get `ERROR_CODE_BUSY`. This
gives the first-connected phone an uncontended tuning window for
multi-client sharing. Clients handle `ERROR_CODE_BUSY` by reading
`GetRadioSettings` and offering the user "[Use radio's] / [Apply mine]" after
the window — including when the refusal was a TX in flight rather than the
lock, since both are resolved by retrying. Device settings have no such
lock: identity is a once-per-device decision and last-writer-wins is
self-healing.

## 7. Wi-Fi station (ESP32 boards only)

- **Lifecycle.** Station mode only, one network, DHCP, applied from the
  device settings. Enable → connect; disable → disconnect and close the
  server; credential change while enabled → reconnect. The TCP port can
  change while up, which rebinds the listener and drops the sockets that
  were on the old port (a client reconnects and resumes from its cursor). A
  pigeon with Wi-Fi enabled comes back on the network after a reboot with no
  app attached.
- **Backoff.** Retries run on an exponential backoff — 5 s doubling to a
  2 min cap, forever — and the firmware is the *only* retry authority: the
  Arduino core's own auto-reconnect is disabled, because it fires immediately
  on the reasons it considers reconnectable (`NO_AP_FOUND`, `ASSOC_FAIL`,
  `HANDSHAKE_TIMEOUT`, …) and would turn an AP outage into a tight reconnect
  loop regardless of this ladder. A wrong password is not fatal: it surfaces
  as `WIFI_STATE_AUTH_FAIL` in `Status`, and the user may have rotated the
  password. No "disable after N failures" policy exists.
- **Multi-client TCP**, exactly like the ESP32 BLE sink: one frame reader and
  one sink per socket, broadcast to all of them. The sink registry holds **8
  slots shared by every transport** — USB CDC, the three BLE centrals the
  ESP32 sink admits, and the four TCP clients below, which is exactly the
  budget. The accept path turns a client away rather than accepting a
  connection nothing will ever be sent to. Default port 5000 (the convention
  MeshCore's desktop tooling standardized on), overridable with `wifi_port`.
  A socket that cannot take a whole frame is closed: a client either sees the
  whole stream terminated by `FetchEnd`, or a disconnect it can resume from
  its cursor.
- **mDNS.** The device advertises `<hostname>.local` with a `_meshpigeon._tcp`
  service on the current port, so desktop tooling finds the pigeon without
  typing an IP. The hostname shares the derived device-name suffix
  (`meshpigeon-A3F2`).
- **No TLS, no cloud, no outbound connections.** The firmware listens; it
  never dials.

## 8. Device settings, the PIN, and the name

`DeviceSettings` is the device-level read model: the name, the Wi-Fi station
settings, and the port. The **PIN is write-only on the wire** — no field of
`DeviceSettingsMessage` carries it, and none ever will; the app remembers what
it set.

### 8.1 `SetDeviceSettings` semantics

- **Atomic.** Every present field is validated first; one bad field rejects
  the whole request with `ERROR_CODE_BAD_PAYLOAD` and **nothing** is applied.
  Partial application invites "which half saved?" UI bugs, and we never want
  a user to believe a setting took when the board can never honor it.
- **Validated.** name ≤ 20 bytes and free of control characters (it is
  advertised verbatim over BLE; bytes ≥ 0x80 are fine, so a non-ASCII name
  works); PIN 4–8 ASCII digits, or the empty string
  to restore the factory default (absent = unchanged); SSID ≤ 32; password
  ≤ 63 (the WPA2/3 limit); port ≠ 0. Every limit is the exact cap in the
  `*.options` files, NUL terminator included — a value at the limit
  round-trips unchanged. Because the cap *is* the limit, a value over it
  never arrives as a bad field: the envelope fails to decode and is dropped
  silently per §2, so there is no request to answer. The firmware's own
  checks are the backstop for whatever does decode.
- **Capability-gated.** Any Wi-Fi field on a board without the `WIFI_STA`
  capability → `ERROR_CODE_NOT_SUPPORTED`, nothing applied. The app hides
  those fields using `DeviceInfo.capabilities`; the firmware rejects as a
  backstop so a stale client cannot leave a dead setting behind.
- **Persists immediately** and applies live: a name change re-advertises, a
  PIN change re-gates immediately, and Wi-Fi settings connect/disconnect.
- **A write that changes nothing is a no-op.** The candidate read model is
  compared against what is already in force first, and a write that lands on
  the same values persists nothing, re-advertises nothing, re-applies nothing
  and pushes nothing — only the issuer's response is sent, because that is
  what it asked for. The persistence inventory is closed and NVS erases wear
  out, so "the client re-sent its settings" must not cost a flash write; and a
  same-value PIN write is not a rotation, so it does not de-authorize the
  other connections (§8.2).
- **No first-owner lock** (§6.1).
- Accepted writes broadcast the full post-write `DeviceSettings` to all
  *other* clients, then answer the issuer with the same read model. That
  broadcast — the only async push that carries a credential — reaches only
  connections that could have called `GetDeviceSettings` anyway; an
  unauthenticated socket attached to the device learns nothing from it.

### 8.2 The PIN

The node holds no identity and no keys, so the PIN exists purely to stop
*casual* use: a stranger scanning Bluetooth, or someone on the same LAN who
discovered the TCP port. It is a protocol-level gate — not BLE pairing —
because it must work identically on USB CDC, BLE and Wi-Fi TCP, none of which
have a shared pairing concept.

- **Always set; the default is the public `"0000"`**, exactly like every
  Bluetooth device's default PIN. There is no "no PIN" state: with the
  default in place the node ships open, and a user-chosen replacement is what
  actually protects it. A custom PIN is 4–8 ASCII digits.
- **Exempt operations** (reachable without authenticating): `ping`,
  `get_device_info`, `get_status`, `auth`. Everything else
  answers `ERROR_CODE_AUTH_REQUIRED`. The three async pushes that carry
  protected material (§3) are filtered by the same rule.
- **Per-connection state.** Authenticating on BLE says nothing about the TCP
  socket someone else opened, and a second BLE central says nothing about the
  first; each connection holds its own flag. Changing the PIN is the one
  moment the flags move together, in both directions: the connection that
  wrote the PIN is authorized (it proved it knows the new PIN by writing it —
  otherwise setting a PIN from the shipped default would lock the writer out
  of the device it just configured), and every other connection loses the
  session it held. A session that knew the old PIN does not survive a
  rotation. The flags follow the *link*, not the transport: a BLE central
  that disconnects takes its session with it on both board families, so
  whoever connects next starts outside the gate.
- **Rate limit, per device.** The first 3 failed attempts are free. From the
  4th on, a 1-second penalty window opens, and during it every attempt —
  even with the correct PIN, on any connection — is rejected unevaluated.
  Deliberately *not* per connection: otherwise an attacker just opens
  several sockets and multiplies the guess rate. The cost is that a stranger
  can lock a legitimate client out for a second at a time, which is the
  cheaper side of that trade for a device with no keys. Changing the PIN
  clears the budget.
- A static PIN over plaintext frames stops casual misuse, not a determined
  attacker with a sniffer. That is the right level for a device whose worst
  case leak is packets the *sender* already encrypted.

### 8.3 Name & BLE advertising

The **effective name** is the stored name if non-empty, else
`MeshPigeon-XXXX` where `XXXX` is the BLE address's two low bytes in hex — the
scheme v1 already advertises. The derived default is never persisted; the app
reads the effective name from `DeviceSettings` and prefills it. Writing an
empty name resets to the derived default. A name change renames the GAP
device and restarts advertising: live connections are unaffected, scanners
see the new name at the next advertising round.

The board therefore brings BLE up *before* asking for the effective name and
applies it in the same breath: the address the suffix comes from does not
exist until the radio's own stack has initialized, and asking first would
advertise `MeshPigeon-0000` on every board.

## 9. Status

`GetStatus` is the snapshot; the async `Status` push fires **only on Wi-Fi
state transitions** (no RSSI churn). The three `*_clients` counts are the
per-transport view of who is attached right now — the transport a client is
connected through, so a multi-transport pigeon is never ambiguous about
which link it is using.

| Field | Meaning |
|---|---|
| `wifi_state` | `OFF`, `CONNECTING`, `CONNECTED`, `AUTH_FAIL` (AP rejected us), `ERROR` (no AP / driver) |
| `wifi_ssid` | the SSID we are **associated** with; empty when not associated (the *configured* one is what `GetDeviceSettings` reports, and the two can differ) |
| `wifi_ipv4` | 4 bytes, network byte order; empty when not connected |
| `wifi_port` | the configured TCP port (0 on boards with no Wi-Fi). The listener only runs while associated, so this is the port to *dial*, not a promise that something is listening |
| `wifi_rssi` | dBm; **0 = unknown** (not connected) |
| `ble_clients` | connected BLE centrals |
| `usb_cdc_clients` | 1 while a host has the USB CDC port open (the core's `Serial` truth value: link up on ESP32, DTR on nRF52), else 0 |
| `wifi_tcp_clients` | TCP sockets the Wi-Fi server has open (0 on non-Wi-Fi boards) |

## 10. Packet store

A byte-budgeted RAM ring: entries cost a 16-byte
header plus their exact payload length, and overflow evicts the oldest. A
packet larger than the whole pool is dropped and counted in `dropped`.

- `SendPacket` stores the packet with origin `SENT` and keys up. One TX at a
  time: a second concurrent send answers `ERROR_CODE_BUSY`, as does a
  `SetRadioSettings` (§6); the app's outbox owns retry policy. A radio that is
  not usable (`radio_ok` false) answers `ERROR_CODE_NO_RADIO` and stores
  nothing — the packet is never accepted, so a `TxResult` can never claim a
  transmission that did not happen. Every other send is accepted first and
  then reported: the `TxResult` push follows the response, carrying
  `success = false` if the radio refused the keying up outright and `true`
  once it finishes.
- Packets received on the air are stored with origin `RECEIVED` and pushed
  live to every **authorized** connected client (§3). A packet too large for
  the whole pool is dropped and counted, and gets no live push: the app would
  otherwise see a seq it could never fetch again.
- `FetchPackets` streams every retained entry with `seq > since_seq`, oldest
  first, up to `max_count`, then `FetchEnd` with the delivered count. Both
  zeros mean "everything" — `since_seq = 0` is the start of the boot and
  `max_count = 0` is no bound — because 0 is not a meaningful value for
  either, and a client that asked for "all of them" must not be handed an
  empty `FetchEnd` and read it as an empty store. The firmware clamps a
  non-zero `max_count` to **64 entries per request** (a stream is written
  straight out of the sink inside one handler call, and a board has to stay
  responsive), so `FetchEnd` can come back shorter than asked for: re-issue
  with `since_seq` = the last seq received. The cursor is
  **exclusive** and seq 0 is never assigned, so `since_seq = 0` means "from
  the beginning" and stays correct for the life of the boot.
  `StoreInfo.oldest_seq` is the *gap boundary* instead: everything below it
  has been evicted for good, so compare against it to notice a gap rather
  than starting from it, which would skip the oldest entry still held.
  `since_seq` cursors stay valid across wraps and purges because sequence
  numbers are monotonic and eviction only removes from the oldest end.
- **Board-loop budget.** A transport also decodes at most **16 maximum-size
  frames' worth of bytes per board-loop tick**
  (`FRAME_MAX_DRAIN_BYTES_PER_PUMP`). Without that cap, a client that keeps
  its inbound buffer fed hands the whole loop to itself and the radio stops
  being polled — `Ping` is exempt from the auth gate, so this is reachable
  unauthenticated. The budget is in **bytes, not frames**, because a peer
  streaming pure garbage never completes a frame and a frame counter would
  never advance. It is a throughput ceiling no real client reaches, not a
  rate limit to design around.
- `PurgeStore` clears history only. `FactoryReset` clears the stored device
  settings **and** history, then reboots: "as it shipped". The record is
  deleted rather than overwritten with defaults, so the next boot reads
  "never written" and falls back itself. The radio tuning and the boot count
  survive a factory reset. `StoreInfo.dropped` also survives a purge: it
  counts overflow, which is a running health reading rather than part of the
  history being cleared. Like everything else in the store it is per boot,
  not per device.

## 11. What is persisted

The persistence inventory is **closed** — three things, ever:

| Item | ESP32 | nRF52 |
|---|---|---|
| radio settings | NVS `meshpigeon/radio` | `/radio.bin` |
| boot count | NVS `meshpigeon/boots` | `/boots.bin` |
| device settings (name, PIN, Wi-Fi) | NVS `meshpigeon/dev` | `/dev.bin` |

Device settings are one fixed-layout record on both families, serialized by
`DeviceSettings::serialize` in the core: a version byte and a trailing CRC16.
One record is what makes a write power-loss tolerant, as the `SettingsStore`
contract requires: a cut leaves the old record or the new one, never a mix of
fields — and the field that goes missing from a mix is the Wi-Fi
configuration, which is exactly what strands a pigeon that came back after
the power went. (NVS replaces a value atomically; on nRF52 the file is
written aside and renamed over the old one.) A record that fails its length,
version or CRC check reads as "never written".
Nothing else is ever written: **no packet bytes, no timestamps, no counters.**
Power loss or reboot clears the packet history — that is a hardware property,
not a policy.

## 12. Semantics the radio does NOT have

Deliberately, per the product's guiding principles:

- No packet parsing, no deduplication, no repeat/rebroadcast logic.
- No ACK logic, no retry logic — the app listens for on-air ACKs through
  the same packet store and owns the retry schedule.
- No keys, no identity, no adverts. A stolen radio leaks nothing.
- No mesh telemetry. The only status fields are device-level.

## 13. Evolution policy (normative)

- **Only ever ADD** fields, oneof variants, or enum values.
- **Never renumber or reuse a field number**; removed ones become `reserved`.
  This binds from the first *shipped* spec: while a `spec_version` is still
  unreleased there is nothing to stay compatible with, so fields may be
  renumbered freely and no `reserved` hole is left behind.
- Unknown fields and unknown oneof variants are skipped by every protobuf
  decoder, so additive changes are backwards compatible by construction. A
  radio that predates a variant answers `ERROR_CODE_BAD_COMMAND`.
- **Additive proto change ≠ `spec_version` bump.** `spec_version` is reserved
  for whole-spec breaks: removing or renumbering a field, changing a field's
  width or meaning, or changing the framing itself. Every bump after 2 ships
  with a paragraph here justifying it.
- Reserved for later, capability-gated, and not implemented today: RTC
  wall-clock, Ethernet, LED/button behaviour, and real ROM/DFU entry for
  `Bootloader` (which today restarts the board). Rejected outright
  (it would give the firmware opinions): anything about packet content,
  identity, or keys.

## 14. Reference

- Firmware: `github.com/meshpigeon/meshpigeon-firmware` (this repo)
- Schema: `protobufs/meshpigeon/*.proto` in this repo
- App implementation: `meshpigeon-app :core-transport`
- Working on the firmware: `AGENTS.md` (build, layout, conventions, traps)
- Principles: `GUIDING-PRINCIPLES.md`
