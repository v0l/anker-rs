# anker-rs

A tiny Rust library (`anker_solix`) and CLI (`anker`) for monitoring and
controlling **Anker SOLIX** portable power stations over **Bluetooth LE** — no
cloud, no account, and no 200 MB app.

Confirmed working on real hardware against a **SOLIX C1000 Gen 2** (A1763):
telemetry read, AC/DC output control, and the full encrypted handshake.

## Features

- 🔍 Scan for nearby SOLIX devices
- 🔐 Full ECDH (secp256r1) + AES-128-CBC session negotiation (matches app firmware)
- 🔋 Battery %, health, charge limits, temperature
- ⚡ Power in/out and per-port status/watts (AC, DC 12 V, solar, USB-C/A)
- 🎛️ Toggle AC and DC outputs
- 🧾 Serial / part number / firmware
- 📤 Text or JSON output

## Supported models

| Model | Telemetry | Control | Notes |
|-------|-----------|---------|-------|
| C1000 Gen 2 (A1763) | ✅ tested | ✅ tested | Needs `4100` subscribe; telemetry on `c421`/`c900`; AC `4101`, DC `4102` |
| C300 / C800 / C1000 gen 1, F2000, F3800 | ⚠️ implemented | ⚠️ implemented | Gen-1 layout; not yet verified here |

Model is auto-detected from the advertised BLE name.

## Build

```bash
cargo build --release
```

> **macOS:** grant your terminal app Bluetooth permission
> (System Settings → Privacy & Security → Bluetooth) or scans return nothing.
> macOS also masks the MAC address, so target devices by name.

## Usage

```bash
# List nearby stations
anker scan

# One telemetry snapshot (targets a name substring by default: "C1000")
anker status
anker -d "C1000" status
anker -f json status

# Live stream until Ctrl-C
anker monitor

# Control outputs
anker dc on
anker ac off
```

Global flags: `-d/--device <name|mac>`, `--scan-secs <n>`, `-f/--format text|json`,
`-v` / `-vv` for logging.

## Library

```rust
use anker_solix::Device;
use std::time::Duration;

#[tokio::main]
async fn main() -> anker_solix::Result<()> {
    let mut dev = Device::find_and_connect("C1000", 6).await?;
    let t = dev.next_telemetry(Duration::from_secs(10)).await?;
    println!("battery {:?}%", t.battery_percentage);
    dev.set_dc(true).await?;
    Ok(())
}
```

## How it works

1. **Discover** — scan for the advertised identifier service `0000ff09-…`.
2. **Negotiate** — a fixed 6-stage handshake on the command characteristic
   (`8c85_0002-…`). The client private key is fixed, so every outbound frame is
   constant; the station returns an ephemeral public key which is combined via
   ECDH into a 32-byte secret (key = first 16 bytes, IV = last 16).
3. **Session** — telemetry arrives on the notify characteristic (`8c85_0003-…`)
   as `0xFF09`-framed packets, reassembled from fragments and AES-CBC decrypted
   into TLV parameters. Commands are encrypted the same way with a rolling
   timestamp for replay protection.

## Credits & disclaimer

Protocol details build on the prior work in
[flip-dots/SolixBLE](https://github.com/flip-dots/SolixBLE) (MIT). This project
is unofficial and not affiliated with Anker. Use at your own risk.

## License

MIT
