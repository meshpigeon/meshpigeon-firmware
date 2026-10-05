# AGENTS.md — working in `meshpigeon-firmware`

Operating manual for this repository: what it is, what it must never become,
how to build and verify it, where things live, and which traps the codebase
already knows about. **Keep it accurate** — a change that alters the interface,
the layout, the build, or the rules below updates this file, `README.md`, and
`docs/radio-protocol.md` in the same commit.

Historical design plans used to live in `docs/plans/`. They are gone: what was
durable now lives here and in `docs/radio-protocol.md`, and what was not
(proposals, brainstorms, superseded wire encodings) is deliberately forgotten.
Do not recreate a `docs/plans/` directory; `docs/` holds supplemental
documentation only.

---

## 1. What this firmware is

**The dumbest possible durable LoRa radio.** It receives packets it cannot
read into the largest memory it can hold, stamps each with uptime, keys up on
demand, and persists its radio settings. It speaks **no mesh protocol** and
holds **no keys** — all of that lives in the app
(`github.com/meshpigeon/meshpigeon-android`).

### The one review gate

> If a proposed change gives the firmware opinions about repeating, message
> content, keys, or identity — stop; it belongs in the app.

Concretely, none of these concepts exist in this codebase, and adding one is a
design change, not a feature:

- no packet parsing beyond a byte count (raw bytes are all the store keeps),
- no dedup table, no repeat mode, no ACK/retry logic,
- no identity, no adverts, no keys, no mesh telemetry.

The **only** opinionated things the firmware holds are the persisted radio
settings and the device settings (name, PIN, Wi-Fi) — both of which exist to be
*configured* by an app, not to act on their own. The app-driven "repeater" is an
app feature: the app decides what to forward and calls `SendPacket`.

The persistence inventory is **closed**: radio settings, boot count, device
settings. Nothing else is ever written to flash — in particular **no packet
bytes, no timestamps, no counters** (§10, §11 of the protocol doc). A stolen
radio must have no history to recover, and a busy mesh must not burn flash
erase cycles on data the design says the device should not keep.

`GUIDING-PRINCIPLES.md` holds the product-level principles; this section is
the firmware-specific addendum and wins when they conflict.

---

## 2. The device interface (v2)

**The `.proto` files are the spec** — `protobufs/meshpigeon/{envelope,device,
radio}.proto` plus the `*.options` size caps. `docs/radio-protocol.md` is their
prose companion (framing, semantics, evolution policy) and is normative too.
`DeviceInfo.spec_version` is **2**.

```
wire frame := COBS( envelope ‖ crc16 ) 0x00        CRC-16/CCITT-FALSE, LE
envelope   := serialized ClientToRadio or RadioToClient
```
- Requests carry a non-zero `id`; responses echo it; async pushes use `id = 0`.
  An envelope that fails to decode, or decodes with `id == 0`, is **dropped
  silently** — there is no id to answer an error against.
- The `body` oneofs enumerate the operations. Unknown oneof variants answer
  `ERROR_CODE_BAD_COMMAND`; unknown *fields* are skipped by the decoder, so
  additive proto changes need no version bump.
- Frame payload cap 512 B; a raw on-air packet is ≤ 255 B (SX12xx silicon
  cap). `PacketEntry.uptime_ms` and `DeviceInfo.uptime_ms` are 64-bit.
- Slow-changing health readings live in `DeviceInfo` (battery, `radio_ok`,
  `noise_floor_dbm` — the last packet's RSSI − SNR, 0 = unknown); live
  connection state lives in `Status` (Wi-Fi link plus the per-transport
  `ble_clients` / `usb_cdc_clients` / `wifi_tcp_clients`). Do not put either
  kind in the other message.
- The PIN is **write-only on the wire** — `DeviceSettings` has no PIN field
  and must never grow one. Writing an empty `pin` restores the factory
  default; absent leaves it alone.
- Device settings: name, Wi-Fi (enabled/ssid/password/port, default 5000).
  `SetDeviceSettings` is atomic (validate everything, then apply) and
  capability-gated (Wi-Fi on a non-Wi-Fi board → `NOT_SUPPORTED`, nothing
  applied). Auth gates it, the PIN, radio settings, and the store commands;
  ping, device info, status and auth stay open.
- Auth is per connection, the shipped default PIN is the public `"0000"`, three
  failures are free and then a 1 s penalty window applies (per *device*, not
  per connection, so parallel sockets do not multiply the guess rate). The
  three async pushes that carry protected material — `DeviceSettings` (the
  Wi-Fi passphrase), the live `PacketEntry`, and the `RadioSettings` re-tune
  — are filtered by the same `is_authorized()` question, because a push that
  an unauthenticated socket can read makes the gate on the request
  pointless. A push that is not the response to a gated request
  (`TxResult`, `Status`) stays open.
- Wi-Fi (ESP32 envs only): station + DHCP, multi-client TCP (≤ 4), mDNS
  `meshpigeon-XXXX.local`, retries 5 s → 2 min forever.

Full semantics: `docs/radio-protocol.md`. Keep the two in sync — the doc is
what a client author reads.

### Adding an operation

1. Add the request/response messages and the `oneof` members in the `.proto`
   files (**append only** — never renumber, never reuse; `reserved` what you
   remove). That binds from the first *shipped* spec: while `spec_version` is
   still unreleased there is nothing to stay compatible with, so renumber
   freely and leave no `reserved` hole.
2. Extend `protobufs/meshpigeon/*.options` if the new field is a string/bytes
   field: **the generated C must stay 100 % static** — no `pb_callback_t`, no
   heap. A field without a cap silently reintroduces them. A **string** cap is
   the documented character limit **+ 1** (nanopb counts the NUL inside the
   buffer, so `max_size: 20` carries only 19 characters — a cap that looks
   right and truncates every maximum-size value); a **bytes** cap is the
   exact size. A cap on a `repeated` field is a `max_count`, and the constant
   the core hands a board hook is *derived* from the generated array
   (`kMaxCapabilities`), so the two cannot drift apart.
3. Regenerate: `NANOPB_DIR=/path/to/nanopb-0.4.9 ./scripts/regen-protos.sh`
   and commit the result. CI fails on stale output.
4. Handle it in `CommandProcessor::handle_request`, with auth where the
   protocol doc says so.
5. Cover it in `test/test_core.cpp`; update `docs/radio-protocol.md` (and
   `AGENTS.md` if it changes the shape of the interface).

`spec_version` is **not** bumped for additive changes — only for removals,
renumbering, width/meaning changes, or a framing change.

---

## 3. Layout

```
protobufs/meshpigeon/        the wire spec: .proto + .options caps
buf.yaml                     `buf lint` / `buf breaking` config
lib/meshpigeon-core/         board-neutral core (built for boards AND host)
  include/meshpigeon/        protocol.h, framing.h, frame_drain.h, settings.h,
                             packet_store.h, uptime_clock.h, command_processor.h,
                             proto_alias.h
  src/                       the .cpp files + generated/ + nanopb/
  src/generated/             COMMITTED nanopb output (regen-protos.sh)
  src/nanopb/                vendored nanopb 0.4.9 runtime
src/main.cpp                 board main: radio + transports + persistence + hooks
src/radio_sx1262.h           RadioLib SX1262 port (raw bytes only)
src/radio_lr1110.h           RadioLib LR1110 port (T1000-E)
src/transports.h             USB CDC + BLE (Nordic UART Service) frame sinks
src/wifi_transport.{h,cpp}   Wi-Fi station + multi-client TCP + mDNS (ESP32 only)
src/sim_main.cpp             desktop radio simulator (TCP, scriptable RF loss/dup)
include/meshpigeon/sim_radio.h  SimRadio : ILoRaRadio for the simulator
test/test_core.cpp           host unit tests (Unity), the whole core
boards/, variants/           custom board definitions + linker scripts
scripts/                     regen-protos.sh, bench.sh, mesh-sim.sh
docs/                        supplemental documentation only
  radio-protocol.md          the versioned interface contract
```

Board-neutral logic belongs in `lib/meshpigeon-core/` so it compiles and is
tested on the host. `src/` is board glue: RadioLib, the three transports, and
the board's persistence.

---

## 4. Build, test, verify

```sh
pio test -e native        # the host tests — the whole core, no hardware
pio run                   # all four board targets must build
pio run -e sim            # desktop simulator (.pio/build/sim/program)
scripts/mesh-sim.sh 3     # N simulated radios on ports 8801..8803
NANOPB_DIR=/path/to/nanopb-0.4.9 ./scripts/regen-protos.sh [--check]
```

Envs: `xiao_wio`, `heltec_v3` (ESP32-S3, USB CDC + BLE + Wi-Fi TCP), `t114`,
`t1000e` (nRF52840, USB CDC + BLE), plus `native` (unit tests) and `sim`.
`pio run -e native` fails **by design** ("Nothing to build" — the env's
`build_src_filter` is `-<*>`); that is not a bug to fix.

`buf lint` / `buf breaking` are configured (`buf.yaml`) but not wired into CI;
installing buf in the runner is a small, welcome addition. `regen-protos.sh
--check` **is** the CI gate on the schema. The nanopb generator is **not**
vendored: fetch the 0.4.9 `linux-x86` tarball somewhere and point `NANOPB_DIR`
at it (CI does this into `$HOME/nanopb`).

The simulator is the development loop for anything protocol-shaped: it speaks
the identical core over TCP, so a full app pipeline can be exercised with zero
hardware. `--fake-wifi` scripts a `OFF → CONNECTING → CONNECTED` status
machine for app CI. `scripts/bench.sh` is the (still skeletal) hardware rig
driver. A *script* driving it must mirror the framing itself (§6, last trap);
there is no scripted client in the repo, so if you add one for app CI, put it
in `scripts/` rather than re-deriving the wire format each time.

### CI

`.github/workflows/ci.yml`: `host-tests` (native tests + sim build + the
protobuf regen gate, with nanopb fetched to `$HOME/nanopb`), `boards` (matrix
build of the four targets), `bench` (self-hosted runner, `bench-ready` label).

---

## 5. Conventions

- **Style**: 4-space indent, `snake_case` functions/locals, `PascalCase`
  types, doxygen-ish `/** */` on public APIs, `NULL` (not `nullptr`) in this
  codebase. Match the file you are in; do not reformat unrelated lines.
- **Minimal diffs**: every change maps to a stated need. No drive-by refactors,
  no dependency upgrades smuggled into a fix.
- **Commits**: conventional commits (`feat:`, `fix:`, `docs:`, `test:`,
  `chore:`), one logical change each.
- **Tests**: every behavior change adds or extends a test in `test/`, named
  for the requirement it protects. The host tests are cheap — run them.
- Board ports never `#include` Arduino headers into the core; the core never
  sees a board.

---

## 6. Traps this codebase already knows about

These are not hypotheticals; each one cost a debugging session.

- **Never turn Wi-Fi modem sleep off while BLE is up.** `WiFi.setSleep(false)`
  is `esp_wifi_set_ps(WIFI_PS_NONE)`, and ESP-IDF's coexistence layer *aborts
  the chip* when that is called with the BT controller already running — which
  it always is here, because `setup()` starts the BLE sink before the Wi-Fi
  transport (the derived name needs the BLE address). It is not a link failure
  and not a retryable one: the settings were already persisted, so the abort
  happens again on **every boot**, and because `apply_wifi()` runs before
  `send_device_settings()` the client never even gets an error — its request
  times out having been applied. The only ways out are erasing the settings
  record or reflashing, and every operation that could write that record is
  itself auth-gated. `WIFI_PS_MIN_MODEM` (the core default) is what the
  coexistence path requires.
- **The USB CDC RX queue is 256 bytes unless you say otherwise.** `Serial` on
  ESP32 is the core's `HWCDC`, whose queue defaults to 256 and whose ISR
  **silently** drops the rest of a 64-byte chunk that does not fit — no log
  anywhere, and a truncated frame is then dropped exactly as a CRC failure is,
  so the client cannot tell. At the default, a host that burst-writes a frame
  over ~256 wire bytes loses bytes, and a legal 255-byte `SendPacket` (268-byte
  frame) landed about 10% of the time. `main.cpp` calls
  `Serial.setRxBufferSize(FRAME_MAX_WIRE + 64)` before `begin()`;
  `HWCDC::begin()` only applies its default when no queue exists yet, so the
  call has to come first.
- **This port is not a console, and the ESP-IDF logs are not silenced by
  `setDebugOutput`.** That call gates the *Arduino* log handler only; the Wi-Fi
  driver's own `ESP_LOGE` still lands on the same byte stream. A log line
  contains no `0x00`, so a client buffers it as a frame body until the next
  frame's delimiter — consuming that frame. The observable effect is **one
  response silently lost per log line, with the request's side effect still
  applied** (observed ~1 Wi-Fi teardown in 25). A log line containing a `0x00`
  would desynchronise the stream outright. The bench suite retries
  un-answered requests and checks `boot_count` rather than re-sending a restart
  op; anything else must assume a lost ack is possible.
- **The CRC is not optional.** `CommandProcessor` hands sinks a bare envelope.
  Transports must call `frame_encode_envelope()` (which appends the CRC before
  COBS) — using `frame_encode_wire()` directly sends un-CRC'd frames and every
  receiver silently drops them. `FrameReader::feed()` returns the envelope
  length *without* the CRC, so `res` is the payload length, not the frame
  length.
- **Bandwidth units are 0.01 kHz internally, plain Hz on the wire** — the
  conversion is `×10` / `÷10`, not `×100` / `÷100`. `RadioSettings` still
  carries a `region` field for on-flash format compatibility; new writes always
  set 0 (the region-preset concept is gone).
- **A field is range-checked at the wire or it is not checked at all.** Every
  value the core narrows on its way to the hardware is validated against the
  *narrowed* range, not merely the incoming type: `power_dbm` is a `uint32`
  on the wire but reaches the radio as an `int8_t`, so 200 would arrive as
  −56 dBm and key up at the wrong power instead of failing
  (`MESHPIGEON_TX_POWER_MAX`, §2). `bandwidth_hz` is the same shape (checked
  before the ÷10 conversion, so 655370 Hz cannot wrap into 10 Hz). A new
  narrowing cast needs its range check in the same commit.
- **A discarded return value is dead code, and the compiler will take you at
  your word.** `DeviceSettings::deserialize()` and
  `RadioSettings::deserialize()` both start with `*this = defaults();` so a
  *rejected* record still leaves a valid object behind — a caller that
  ignores the return value then has the shipped defaults rather than stack
  garbage with an unterminated PIN in it. Written as a bare `defaults();`
  statement instead, GCC deleted the call (nothing observes the returned
  temporary) and the guarantee silently evaporated; only
  `test_device_settings_record_roundtrips`, asserting that a *truncated*
  record reads back as defaults, caught it. Assign through the result. This
  is the same lesson as the brace-assignment trap below, one level up.
- **`WiFiServer::available()` is the accept, not a probe.** On the pinned
  Arduino core it *returns a `WiFiClient`*, and `accept()` is literally
  `return available();`. So `if (!server.available()) return; WiFiClient c =
  server.accept();` accepts the pending connection into a temporary
  `WiFiClient` — whose destructor closes the socket — and the second call
  then finds nothing: every TCP client is accepted and instantly hung up on
  and the slot array never fills. It compiles clean, because the returned
  client is contextually convertible to `bool`. One `accept()` is both the
  probe and the accept: it returns an empty client when nothing is pending.
- **COBS is defined over a byte *stream*, so the encoder is a state
  machine.** `CobsEncoder` (put/finish) is the primitive; `cobs_encode()` is
  it used once, and `frame_encode_envelope()` uses it again for the two CRC
  bytes, so no frame is ever assembled in a scratch buffer. Do not
  reintroduce the scratch frame: it was 516 bytes of stack in the deepest
  call chain on a board whose loop task has 4 KB of it (`LOOP_STACK_SZ` in
  the nRF52 BSP). For the same reason `IFrameSink` owns its own `wire_`
  buffer behind `encode_wire()`: a `FRAME_MAX_WIRE` local in each of the four
  `send_frame()` implementations was the largest single frame on that stack,
  and RAM is the plentiful resource on every board that builds this.
- **Both LoRa radio ports share one state machine** (`src/radio_lora.h`,
  `LoraRadioBase<Radio, Traits>`); the two `radio_*.h` files carry only
  `begin()` and a `Traits` struct of the three facts that differ (IRQ bit
  names, on-air CRC length, the LR1110 header-error quirk). Net flash is
  slightly *smaller* than the two hand-written copies were. Keep the trait
  members `constexpr`: they are read in `if` conditions, and a plain
  `static const` member is not a constant expression. Do not re-introduce a
  second copy of the TX/RX loop.
- **Uptime wrap detection is `now < last_ms_`**, nothing fancier. A monotonic
  32-bit `millis()` only goes backwards at the wrap; adding a "is it a big
  enough jump" guard makes the real case (wrapping *into a small value*)
  undetectable. `CommandProcessor::poll()` polls the clock, and board loops
  poll it too.
- **nanopb `SetRadioSettings.settings` is an optional submessage**: setting the
  field without `has_settings` encodes an empty message — and *omitting* it
  decodes as an empty one too, indistinguishable from "all defaults" unless
  the handler checks `has_settings` itself. Same for every `optional` field
  in `SetDeviceSettings` — set the `has_*` flag.
- **A nanopb string `max_size` counts the NUL.** `max_size: 20` is a
  19-character field, so a cap that matches the documented limit by eye
  truncates every value *at* the limit. Caps are limit + 1;
  `test_max_length_strings_survive_the_wire` round-trips each one through real
  encode/decode, so a future edit fails there, not on a device. The corollary:
  because the cap *is* the documented limit, a value over it fails to decode
  and is dropped silently (§2) — it never reaches the firmware as a bad field
  the validation could answer with `BAD_PAYLOAD`.
- **COBS is not length-preserving, and the wire buffer is the bigger one.**
  `FrameReader` buffers up to `FRAME_MAX_WIRE` (518) bytes but hands out
  `FRAME_MAX_DECODED` (514), and a COBS body expands by up to 3 bytes over
  the decoded form — so an unbounded `cobs_decode` writes straight past every
  transport's frame buffer (on the nRF52 sink, into the TX queue's `head_`/
  `count_`). That is why `cobs_decode` takes a `dst_cap`. Any new framing
  primitive that writes into a caller-provided buffer takes a cap too.
- **A hook that fills an array must honour `max`.** `fill_capabilities()` is
  handed `kMaxCapabilities`; a value that does not fit is neither written nor
  counted. Ignoring `max` while still returning the count is how a board ends
  up advertising `CAPABILITY_UNSPECIFIED` for every capability it has.
- **`begin_response()` is the only place a response is reset.** It zeroes the
  whole envelope, union included, so a field a builder (or a board hook)
  that only knows some of the message does not set reads as "unknown"
  instead of as the previous Status's values. Every `build_*`, every
  `send_*` and every inline body fill must call it first — three of them
  used to zero their own body as well, which was pure belt-and-braces (all
  three callers already called `begin_response`) and *misstated* the rule,
  because `send_error`/`send_ok`/`send_pong`/`build_packet_entry` never did
  it. One reset point, one rule, enforced by review rather than by three
  copies of the same block. If a builder is ever added that cannot call
  `begin_response` first, fix the shape instead of adding a second reset. If
  you *do* find yourself needing a second zero, assign through a local
  (`const StatusMessage empty = StatusMessage_init_zero; body = empty;`) and
  not `body.status = StatusMessage_init_zero;` — brace-assignment compiles on
  the host and on arm-none-eabi but the xtensa toolchain rejects it, so
  `pio test -e native` will not catch it and only `pio run` covers all four.
- **`regen-protos.sh` needs an absolute, pre-created output directory.** The
  PyInstaller-packed generator mishandles relative `../` paths. The script
  resolves `OUT` with `$(pwd)` and `mkdir -p`s it; do not "simplify" that back.
- **`src/wifi_transport.cpp` is only in the build for the two ESP32 envs**
  (`build_src_filter = +<main.cpp> +<wifi_transport.cpp>`). A new Wi-Fi-capable
  env needs that line, or the link fails with undefined `WifiTransport`
  symbols.
- **nRF52 BLE TX is queued** (16 frames × `FRAME_MAX_WIRE`). v2 frames are
  ~520 B, so the queue is ~8 KB of RAM; the board still has headroom, but watch
  the `t114`/`t1000e` RAM line when frames grow. `send_frame()` waits a
  *bounded* time for room and then drops the frame: a central that stops
  reading must cost dropped frames, never a wedged board loop.
- **`packet_store.h`'s on-ring `Header` puts the `uint64_t` first** on purpose:
  any other field order pads the struct to 24 bytes and breaks the
  `static_assert(sizeof(Header) == kStoredPacketOverhead)`.
- **Anything the firmware advertises must match the BLE name**: the derived
  default name (`MeshPigeon-XXXX`) and the mDNS hostname (`meshpigeon-XXXX`)
  both come from `BoardHooks::mac_suffix()`. Compute the effective name
  **once** — `CommandProcessor::effective_name()` — and hand it to the
  transport; never re-derive the suffix inside a transport. That is also why
  `main.cpp` boots the core and *then* creates any transport: the settings
  have to be loaded before the first advertisement, and BLE has to be up
  before the name can be derived.
- **The `CommandProcessor` sink registry** holds **8** clients total, shared
  across every transport (`kMaxSinks`) — exactly USB CDC + the ESP32 BLE
  sink's three centrals + the four TCP clients the doc promises, so the two
  limits cannot disagree. `add_sink()` returns `false` when it is full, so a
  transport turns the extra client away; never ignore the result. The ESP32
  BLE sink holds a **fixed array** of connections, not a vector: the registry
  stores pointers into it, so a reallocation would leave every registered
  connection dangling.
- **Auth state lives on the connection, so a transport with more than one
  connection needs more than one sink.** The ESP32 BLE sink used to be one
  sink for all centrals, on the reasoning that a NUS link is "one serial
  console" — which is false the moment a second central attaches, and made
  it possible for a second central to inherit the session the first one had
  authenticated. Each central is now its own `IFrameSink`, and notifies only
  its own central (`ble_gatts_notify_custom` with its connection handle —
  `NimBLECharacteristic::notify()` sends to *every* subscribed central, which
  would deliver a response past another connection's auth gate). A push is one
  serialization either way.
  The flip side is that with **one** sink (the nRF52 boards) the sink *is*
  the link, so `pump()` has to clear `authenticated` when it drops: an
  ESP32-style per-connection sink gets that for free, a static one does not,
  and without it whoever connects to a T114 after an authenticated session
  starts inside the gate.
- **The derived name needs the BLE stack, so BLE comes first.**
  `effective_name()` derives `MeshPigeon-XXXX` from the BLE address, and on
  both families that address does not exist until the stack is up
  (`NimBLEDevice::init()` / `Bluefruit.begin()`). Asking first advertised
  `MeshPigeon-0000` — and, on nRF52, `MeshPigeon-` with no suffix at all,
  because `sd_ble_gap_addr_get()` simply fails. Hence `BleSink::begin()`
  takes no name and does not start advertising; `set_name()` is the one place
  a name is ever set, for the first advertisement and every rename alike.
- **The board loop is the only task that may touch the core.** NimBLE-Arduino
  runs the BLE host in its own FreeRTOS task at `configMAX_PRIORITIES-4`,
  which *preempts* the Arduino `loop()` (priority 1), and the GATT write
  callback runs there. The core is single-threaded by design — one
  `response_` scratch, one sink registry — so a callback that calls
  `on_envelope()` races the tick that is polling the radio: the loser is a
  half-built envelope broadcast to every transport, or a registry walked
  while it is being mutated. Every transport therefore does its inbound
  work in `pump()` and its callbacks only copy bytes into a FIFO (or, on
  nRF52, into `BLEUart`'s), and a callback that changes the *set* of
  connections only flags it for the loop. There is no lock in the core, by
  design — do not add one, move the work instead.
- **An ack for a restart has to leave the building before the building
  does.** `Reboot` / `FactoryReset` / `Bootloader` answer `Ok` and *then*
  ask the board to reset, and on both BLE families a frame is queued rather
  than handed to the link — so `reboot_now()` calls `BleSink::flush()` on
  the way out. Bounded on purpose: a central that stopped reading must cost
  that delay and nothing more.
- **Never re-tune the radio while it is transmitting.** The tuning sequence
  calls `standby()`, which aborts the TX on the silicon: the transmission
  never completes, `tx_done()` never sees `kIrqTxDone` again, and the node
  answers `BUSY` to every `SendPacket` until it is power-cycled. The core
  rejects the retune with `ERROR_CODE_BUSY` and `LoraRadioBase::apply()`
  refuses as a backstop.
- **A sink's `send_frame()` must call `encode_wire()` first.** `wire()` is only
  meaningful after it; `Serial.write(wire(), len)` compiles clean and sends
  whatever the last frame left in the buffer (length being the *envelope's*,
  not the wire form's). The USB CDC sink shipped that way once.
- **`NimBLECharacteristic::notify()` is a broadcast.** It sends to every
  subscribed central, so per-connection sinks that call it deliver each
  response to every connection (past its auth gate) and each push N times. Use
  `ble_gatts_notify_custom(conn_handle, …)` and honour the CCCD yourself
  (`onSubscribe`). The controller also stops advertising when a central
  connects, so the second and third are only reachable if `pump()` restarts it.
- **A rename is not applied by `start()` alone.** NimBLE's `start()` returns
  early while advertising is active (the payload is never rebuilt), and
  Bluefruit's scan response holds a *copy* of the name taken when `addName()`
  ran. Both `set_name()`s stop advertising and rebuild the payload first.
- **`WiFiClient::write()` blocks the caller for up to 10 s** on a peer whose
  receive window is full — the caller is the board loop. The TCP sink writes
  with `MSG_DONTWAIT` under a short budget and marks the socket broken
  instead. Accepted sockets also get TCP keepalive: a peer that vanished
  without a FIN would otherwise hold one of the four slots indefinitely.
- **`DeviceSettings::set_pin()` must pad, not just copy.** A `memcpy` of the
  new PIN's length leaves the tail of a longer old one in place (`12345678` →
  `4321` reads back as `43215678`). The field is zeroed first.
- **`FetchPackets` is capped per request** (64 entries). The stream is written
  straight out of the sink inside one handler call, so an unbounded
  `max_count` would let a client hold the loop (and the BLE queue) for as long
  as the store is deep. `FetchEnd` can report fewer entries than asked for;
  clients resume from the last `seq`. The same reasoning bounds **inbound**
  decoding: a transport decodes at most `FRAME_MAX_DRAIN_BYTES_PER_PUMP`
  (16 max-size frames' worth) per board-loop tick, because `Ping` is
  auth-exempt and an unbounded `while (available())` would let one client keep
  the radio from ever being polled. The budget is in **bytes, not frames** —
  garbage never completes a frame, so a frame counter would never advance.
  Every transport that drains a socket/FIFO in `pump()` calls
  `drain_frames()` (`frame_drain.h`) rather than writing the loop out again:
  the budget is a safety property, and it is exactly the kind of rule that
  gets forgotten in the fourth hand-written copy.
- **Client slots are not packed.** A client that disconnects leaves a hole,
  so every walk over a fixed client array must go by slot index, never by
  "the i-th live client" — `stop_server()` looping
  `while (num_clients_ > 0) drop_client(num_clients_ - 1)` hung the board
  forever the moment a hole sat at that index.
- **`MESHPIGEON_MAX_FRAME_PAYLOAD` is asserted against nanopb's own size
  estimate** in `proto_alias.h` (`meshpigeon_RadioToClient_size` /
  `meshpigeon_ClientToRadio_size`). Grow a field, a cap or a oneof tag past
  the cap and the *build* fails, instead of the answer silently vanishing on
  the device because `encode_response()` returned 0.
- **The CRC is on the wire, so any client that isn't a transport has to strip
  it.** What arrives is `COBS(envelope ‖ crc16) 0x00`: read to the `0x00`
  delimiter, COBS-decode, **drop the trailing 2 bytes**, *then* protobuf-decode.
  A `FrameReader` does all of this for you (`feed()` hands back the envelope
  only) — the step only bites hand-rolled probes against the sim, where
  parsing a frame that still carries its CRC fails with a nonsense wire type
  several fields in.
