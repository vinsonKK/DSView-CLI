# dsview-cli Usage

`dsview-cli` is DSView's headless command-line bridge. It calls libsigrok4DSL
directly to control DreamSourceLab USB instruments (DSLogic logic analyzers,
DSCope oscilloscopes). No Qt dependency.

This document describes the interface of **the CLI itself**. For the tool
interface of the upper-layer MCP server (`dsview_mcp.py`), see
[`CLAUDE.md`](CLAUDE.md) and [`README.md`](README.md).

---

## 1. Build

### Linux

```bash
sudo apt install build-essential cmake pkg-config \
    libglib2.0-dev libusb-1.0-0-dev zlib1g-dev python3-dev
cd /path/to/DSView
cmake -S . -B build
cmake --build build --target dsview-cli -j$(nproc)
```

### Windows (MSYS2 MINGW64)

```bash
pacman -S --needed mingw-w64-x86_64-{gcc,cmake,ninja,pkgconf} \
    mingw-w64-x86_64-{glib2,libusb,zlib} \
    mingw-w64-x86_64-{fftw,boost,qt5-base}
cd /i/code/DSView
cmake -S . -B build -G Ninja -DCMAKE_BUILD_TYPE=RelWithDebInfo \
    -DCMAKE_POLICY_VERSION_MINIMUM=3.5
cmake --build build --target dsview-cli -j$(nproc)
```

Notes:

- `fftw` / `boost` / `qt5-base` are only needed to satisfy the dependency
  checks in the top-level `CMakeLists.txt`; the CLI itself does not link
  against them.
- `-DCMAKE_POLICY_VERSION_MINIMUM=3.5` works around CMake 4 rejecting
  `cmake_minimum_required(VERSION 2.8.6)`.
- Runtime DLLs: `libglib-2.0-0.dll`, `libwinpthread-1.dll`, `zlib1.dll`,
  `libusb-1.0.dll`, all located in `C:\msys64\mingw64\bin`. Copy them
  manually if you run the binary outside that directory.

### Output location

On every platform the binary is written to `build.dir/` inside the source tree:

```
build.dir/dsview-cli        (Linux)
build.dir/dsview-cli.exe    (Windows)
```

### Firmware path

Opening a device requires FPGA firmware. The CLI searches the following
locations in order, all relative to the directory of the executable:

1. `<exe_dir>/../DSView/res`
2. `<exe_dir>/../../DSView/res`
3. `/usr/share/DSView/res` (system install path)

Running `build.dir/dsview-cli` from inside the source tree hits entry 1.

---

## 2. General conventions

| Item | Description |
|------|-------------|
| stdout | JSON results only; can be piped straight into `jq` |
| stderr | libsigrok runtime log (`sr: ...`); discard or redirect when parsing |
| Exit code | `0` success, `1` failure |
| Help | `dsview-cli` (no arguments) or `dsview-cli <subcommand> -h` |

`dsview-cli --help` is **not available** — `--help` is treated as a
subcommand name and reports `Unknown command`.

Numeric arguments accept SI suffixes (case-insensitive): `k`=1e3,
`M`/`m`=1e6, `G`/`g`=1e9. For example `-s 10M`, `-n 100k`, `-s 1G`.

---

## 3. Subcommands

### 3.1 `scan` — list devices

```bash
dsview-cli scan
```

Actual output:

```json
[
  {"index": 0, "handle": 2752036612528, "name": "Demo Device"},
  {"index": 1, "handle": 2752061933216, "name": "DSLogic PLus"}
]
```

`index` is the value to pass to `-d` later. `Demo Device` is an always-present
software-simulated device, useful for validating the whole capture chain
without hardware.

### 3.2 `info` — query device capabilities

```bash
dsview-cli info [-d N]
```

Actual output (DSLogic Plus, excerpt):

```json
{
  "index": 1,
  "name": "DSLogic PLus",
  "driver": "DSLogic",
  "dsview_version": "1.3.2",
  "channels": 16,
  "channel_range": [0, 15],
  "mode": 0,
  "mode_name": "LOGIC",
  "samplerate": 1000000,
  "limit_samples": 1000000,
  "vth": 1.00,
  "channel_modes": [
    {"id": 0, "desc": "Use 16 Channels (Max 20MHz)"},
    {"id": 1, "desc": "Use 12 Channels (Max 25MHz)"},
    {"id": 2, "desc": "Use 6 Channels (Max 50MHz)"},
    {"id": 3, "desc": "Use 3 Channels (Max 100MHz)"}
  ]
}
```

`mode_name` determines which set of capture parameters to use:

| `mode` | `mode_name` | Meaning |
|:-:|---|---|
| 0 | `LOGIC` | Logic analyzer |
| 1 | `DSO` | Oscilloscope (buffered mode) |
| 2 | `ANALOG` | Oscilloscope (streaming mode / DAQ) |

`channel_modes` is the channel-count / max-samplerate table for **this
specific device**. It differs between models, so always rely on the actual
`info` output rather than copying a table from another model.

In DSO / ANALOG mode, `info` additionally outputs `analog_channels`
(with per-channel current `vdiv_mV`, `probe_factor`, `coupling`, `bits`,
`hw_offset`) plus `vdiv_options`, `coupling_options`, `probe_factor_options`
and `trigger_types`.

### 3.3 `capture` — acquire samples

```bash
dsview-cli capture [options] -o OUTPUT_FILE
```

#### Common options

| Option | Default | Description |
|--------|---------|-------------|
| `-d, --dev N` | `0` | Device index |
| `-s, --samplerate RATE` | `1M` | Samplerate, e.g. `10M`, `100M` |
| `-n, --samples COUNT` | `1M` | Sample count (per channel), e.g. `100k` |
| `-c, --enable-chs LIST` | all 16 channels | Comma-separated channel numbers, e.g. `0,1,4,7` |
| `-N, --ch-names LIST` | empty | Comma-separated channel names, matched one-to-one with `-c` |
| `-t, --trig-ch N` | `-1` | Trigger channel, `-1` means free-run |
| `-T, --trig-type TYPE` | `none` | `rising`/`falling`/`high`/`low`/`none` |
| `-p, --trig-pos PCT` | `50` | Pre-trigger percentage 0–100 |
| `-o, --out FILE` | `/tmp/dsview_capture.bin` | Output file path |

`-c` **only accepts comma-separated individual channel numbers; range syntax
such as `0-5` is not supported.** Writing `-c 0-5` is parsed by `atoi` as
channel 0 and the remaining channels are silently dropped. Range syntax exists
only in the MCP layer (`dsview_mcp.py`).

#### Logic analyzer only

| Option | Description |
|--------|-------------|
| `-V, --vth VOLTS` | Input voltage threshold 0.0–5.0 V, Pro series only |

Devices without threshold support (including Demo Device) report
`"vth": -1.00` in the result.

#### Oscilloscope only (DSCope)

All three options use the `channel:value` format. The channel number must be
`0` or `1`, and the option is repeated once per channel.

| Option | Allowed values | Example |
|--------|----------------|---------|
| `--vdiv CH:mV` | 10, 20, 50, 100, 200, 500, 1000, 2000 | `--vdiv 0:500 --vdiv 1:1000` |
| `--coupling CH:MODE` | `DC` / `AC` | `--coupling 0:DC --coupling 1:AC` |
| `--probe CH:FACTOR` | 1, 2, 10, 20 | `--probe 0:1 --probe 1:10` |

DSO triggers recognise only `rising` / `falling` / `none`; `high` / `low` are
treated as `rising`.

Voltage conversion formula (8-bit ADC, 0–255, 10 vertical divisions):

```
full_scale_mV = vdiv_mV * probe_factor * 10
voltage_mV    = (hw_offset - raw_sample) * full_scale_mV / 255
```

#### Success output

Actual output (Demo Device, 1 MHz × 100k, channels 0/1):

```json
{
  "success": true,
  "mode": "logic",
  "samples": 99968,
  "samplerate": 1000000,
  "unitsize": 1,
  "vth": -1.00,
  "channel_map": [
    {"seq": 0, "phys": 0, "name": "SDA"},
    {"seq": 1, "phys": 1, "name": "SCL"}
  ],
  "trigger": {"enabled": false, "channel": -1, "type": "none", "pos_pct": 50},
  "file": "...dsv_test.bin",
  "meta": "...dsv_test.bin.meta.json"
}
```

The actual `samples` value may be slightly lower than the `-n` request
(hardware transfers samples in groups of 64; here 100000 → 99968).

#### Failure output

```json
{"success": false, "error": "no devices found"}
{"success": false, "error": "device 1 is in use by another application"}
{"success": false, "error": "failed to activate device N"}
{"success": false, "error": "device init timed out"}
{"success": false, "error": "ds_start_collect failed"}
{"success": false, "error": "cannot open: <path>"}
{"success": false, "error": "capture error or timeout", "samples": N}
```

`device is in use` usually means the DSView GUI has the same device open —
close the GUI and retry.

---

## 4. Output file format

Each `capture` produces two files: `<out>` and `<out>.meta.json`.

### 4.1 `.bin` file header (12 bytes, little-endian)

| Offset | Type | Content |
|:-:|---|---|
| 0 | `uint64` | Samplerate (Hz) |
| 8 | `uint32` | Channel count. For LOGIC this is the channel count of the **hardware channel mode**; for DSO/ANALOG it is the number of enabled channels |

Raw sample data follows the header.

### 4.2 LOGIC sample layout

Parallel format. Each sample occupies `unitsize` bytes (1 when the hardware
mode uses ≤8 channels, otherwise 2), and **bit K corresponds to physical
channel K** (i.e. `phys` in `channel_map`, not `seq`).

```
file size = 12 + samples * unitsize
```

Example: the 99968 samples above × unitsize 1 → 12 + 99968 = 99980 bytes,
matching the measured file size.

The FPGA actually transfers data in cross format (interleaved in groups of
64 bits per channel); the CLI converts it to parallel format internally, so
upper layers need not care.

### 4.3 DSO / ANALOG sample layout

Raw 8-bit ADC values, interleaved per sample:

```
[ch0_s0][ch1_s0][ch0_s1][ch1_s1]...
file size = 12 + samples * enabled_channel_count
```

### 4.4 `.meta.json` sidecar file

Actual output (LOGIC):

```json
{
  "mode": "logic",
  "samplerate": 1000000,
  "samples": 99968,
  "unitsize": 1,
  "channel_map": [
    {"seq": 0, "phys": 0, "name": "SDA", "type": "logic"},
    {"seq": 1, "phys": 1, "name": "SCL", "type": "logic"}
  ],
  "trigger": {"enabled": false, "pos_pct": 50}
}
```

- `mode`: `logic` / `dso` / `analog`; downstream tools pick the decoder from it
- `seq`: position within the `-c` list; `phys`: physical channel number, which
  is also the bit index inside the sample bytes
- In DSO/ANALOG mode each `channel_map` entry additionally carries
  `vdiv_mV`, `probe_factor`, `coupling`, `hw_offset`, `bits`
- When `trigger.enabled` is `false`, the `channel` / `type` fields are omitted

No `.meta.json` is written when the capture fails.

---

## 5. Common examples

```bash
# List devices
dsview-cli scan

# Query capabilities of device 1
dsview-cli info -d 1

# Capture I2C: channels 0/1, 10 MHz, 100k samples
dsview-cli capture -d 1 -s 10M -n 100k -c 0,1 -N SDA,SCL -o i2c.bin

# Capture SPI, trigger on CLK rising edge, 10% pre-trigger
dsview-cli capture -d 1 -s 50M -n 1M \
    -c 0,1,2,3 -N CLK,MOSI,MISO,CS \
    -t 0 -T rising -p 10 -o spi.bin

# 3.3V LVCMOS signals, raise threshold to 1.5V
dsview-cli capture -d 1 -s 10M -n 100k -c 0,1 -V 1.5 -o lvcmos.bin

# Oscilloscope: CH0 500mV/div, DC coupling, 1x probe, rising-edge trigger
dsview-cli capture -d 1 -s 100M -n 10k -c 0 -N VOUT \
    --vdiv 0:500 --coupling 0:DC --probe 0:1 \
    -t 0 -T rising -p 50 -o wave.bin

# Validate the chain with Demo Device when no hardware is attached
dsview-cli capture -d 0 -s 1M -n 100k -c 0,1 -o demo.bin

# Keep only the JSON, drop the libsigrok log
dsview-cli scan 2>/dev/null | jq .
```

---

## 6. Notes and known limitations

**Automatic channel-mode selection (LOGIC)**
The CLI picks the smallest hardware channel mode that covers the **highest
channel number** appearing in `-c`, and programs it into the FPGA. So
`-c 0,1` switches to 3-channel mode (up to 100 MHz on DSLogic Plus), while
`-c 0,15` requires 16-channel mode (up to 20 MHz). To run at a high
samplerate you must keep channel numbers low; reducing only the channel
count while keeping a high channel number has no effect.

**Capture timeout fixed at 120 seconds**
The CLI waits a hard-coded 120 seconds. If sample count ÷ samplerate exceeds
that, it returns `capture error or timeout`. The timeout cannot be adjusted
from the command line (the MCP layer's `timeout` parameter controls the
subprocess timeout, not this one).

**`-c` does not support range syntax**
See section 3.3.

**Mismatched `-N` count is not an error**
If fewer names than channels are given, the extra channels get empty names;
if more are given, the surplus is ignored.

**Default output path on Windows**
The default `/tmp/dsview_capture.bin` is a Linux path. Under a native Windows
exe it resolves to `\tmp\` on the current drive, which usually does not exist,
producing `{"success": false, "error": "cannot open: ..."}`. Always specify
`-o` explicitly on Windows.

**Exclusive device access**
A device can be open in only one process at a time. The CLI cannot capture
while the DSView GUI has the device open, and vice versa.

**Concurrent capture on multiple devices**
Devices with different `index` values are independent, so several CLI
processes can capture simultaneously (e.g. one DSLogic decoding a protocol
while a DSCope watches the analog waveform).
