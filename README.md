<p align="center"><img src="banner.png" width="480" alt="MeshPigeon logo"></p>

# MeshPigeon Radio Firmware

The dumbest possible durable radio: it receives LoRa packets into the largest
memory it can hold, stamps each with uptime, keys up when told, and persists
its radio settings across reboots. **No mesh protocol, no keys, no repeat
logic** — all of that lives in the [MeshPigeon app](https://github.com/meshpigeon/meshpigeon-android).

```
┌──────────────┐   BLE / USB CDC / TCP (protobuf envelopes)
│  MeshPigeon App │ ◄──────────────────────────────────┐
│  (all proto) │                                    │
└──────────────┘                                    ▼
                                      ┌──────────────────────────┐
                                      │ MeshPigeon Radio Firmware   │
                                      │ radio → packet store     │
                                      │ (max packets, uptime)    │
                                      │ + persisted radio settings│
                                      └──────────────────────────┘
                                                   ▼
                                        on-air (MeshCore-compatible
                                        flood/direct routing)
```

## Supported boards

| Board | MCU | Radio | Transports |
|---|---|---|---|
| Seeed XIAO ESP32-S3 + Wio-SX1262 | ESP32-S3 | SX1262 | USB CDC + BLE + Wi-Fi TCP |
| Heltec WiFi LoRa 32 V3 | ESP32-S3 | SX1262 | USB CDC + BLE + Wi-Fi TCP |
| Seeed SenseCAP T114 | nRF52840 | SX1262 | USB CDC + BLE |
| Seeed Tracker T1000-E | nRF52840 | LR1110 | USB CDC + BLE |

Build a board:

```sh
pio run -e xiao_wio        # or heltec_v3, t114, t1000e
pio run -e xiao_wio -t upload
pio run -e xiao_wio -t monitor
```

Run the host-side unit tests (no hardware needed — the entire board-neutral
core is compiled and tested on the desktop):

```sh
pio test -e native         # host unit tests: framing, store, settings,
                           # envelopes, device settings, auth, status, uptime
```

Run the **desktop radio simulator** — a TCP stand-in for a real board that
speaks the identical protocol, for app development and CI with zero hardware:

```sh
pio run -e sim
.pio/build/sim/program --port 8765 --traffic-ms 3000 --loss 10
.pio/build/sim/program --port 8765 --fake-wifi   # scripted Wi-Fi status for app CI
```

## Layout

```
protobufs/meshpigeon/  the .proto files — the wire spec (docs §0)
lib/meshpigeon-core/   board-neutral core: framing (COBS+CRC16), packet store,
                    settings + device settings, uptime clock, command processor
  src/generated/       committed nanopb output (regen-protos.sh; CI checks it)
  src/nanopb/          vendored nanopb 0.4.9 runtime
src/main.cpp        board main (wires radio + transports + core loop)
src/radio_sx1262.h  RadioLib SX1262 port (raw bytes only)
src/radio_lr1110.h  RadioLib LR1110 port (T1000-E)
src/transports.h    USB CDC + BLE (Nordic UART Service) frame sinks
src/wifi_transport.*  Wi-Fi station + multi-client TCP + mDNS (ESP32 envs)
src/sim_main.cpp    desktop simulator (TCP bridge + scriptable RF loss/dup)
include/meshpigeon/sim_radio.h  SimRadio : ILoRaRadio for the simulator
test/               host-side unit tests (Unity, run with -e native)
boards/             custom board definitions (seeed_t114, tracker-t1000-e)
                    + SoftDevice s140 v7 linker script
docs/               radio-protocol.md — the versioned command contract
AGENTS.md           how to work in this repo: build, layout, conventions, traps
```

## Working on this repo

Read [AGENTS.md](AGENTS.md) first: the dumb-radio review gate, the interface
summary, the layout, the build/test commands, and the traps this codebase
already knows about. It is kept in step with this README and
`docs/radio-protocol.md`.

## The interface is protobuf

Every frame carries a serialized `ClientToRadio` / `RadioToClient` envelope
(COBS-framed, CRC-16 over the serialized bytes). The `.proto` files in
`protobufs/meshpigeon/` are the contract; `docs/radio-protocol.md` documents the
framing and the semantics, including the device-settings catalog, the PIN/auth
model, naming, the Wi-Fi lifecycle and the additive-only evolution policy.
See [docs/radio-protocol.md](docs/radio-protocol.md).

## What the firmware never does

- Never decrypts, parses payload types, tracks contacts, stores keys, sends
  adverts, or talks to the internet. It never rebroadcasts on its own —
  if a change gives the firmware opinions, it belongs in the app
  (see [GUIDING-PRINCIPLES.md](GUIDING-PRINCIPLES.md)).

## Acceptance checklist (per board, bench)

- [ ] Boots and listens on persisted settings with no app attached.
- [ ] A fresh board advertises the derived name (`MeshPigeon-XXXX`) and
      resolves `meshpigeon-XXXX.local` — not `MeshPigeon-0000`, which is
      what asking for the name before the BLE stack is up produces.
- [ ] Survives a settings-persistence soak across reboots.
- [ ] Stores ≥ target packet count; overflow drops oldest cleanly.
- [ ] `FetchPackets` replay lets a fresh app reconstruct exact history.
- [ ] A connected app forwards packets; the radio alone never transmits
      without a `SendPacket`.
- [ ] Re-tuning mid-transmission answers `BUSY`, and the radio transmits
      normally again once the send completes.
- [ ] A custom PIN gates the node, and `Auth` unlocks one connection only.
- [ ] Rotating the PIN locks every other connection out again.
- [ ] With a PIN set, an unauthenticated connection sees no live packet
      pushes, no device-settings pushes and no re-tune pushes (and does on
      the default PIN).
- [ ] BLE carries a request while packets arrive off the air and a send
      completes — no garbled or duplicated frame on any transport. (A BLE
      callback runs on NimBLE's own task; everything core-facing happens in
      the board loop.)
- [ ] `Reboot` / `FactoryReset` answer `Ok` *before* the board goes down.
- [ ] USB CDC answers a `GetDeviceInfo` with a frame the host can decode
      (a transport that forgets `encode_wire()` sends stale buffer bytes).
- [ ] With two BLE centrals attached, each receives only the responses to its
      own requests, and one copy of every push.
- [ ] A rename shows up in a scanner without anyone connecting first, on both
      BLE families.
- [ ] A TCP peer that opens a socket, pings and never reads costs the radio
      no more than ~250 ms before its socket is closed.
- [ ] 3 concurrent BLE clients can fetch history simultaneously.
- [ ] Coexists with MeshCore repeaters on-air.

## License

MIT — see [LICENSE](LICENSE).
