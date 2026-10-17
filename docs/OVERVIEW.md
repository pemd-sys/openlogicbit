# open-logic-bit – Architecture Overview

This document is an introduction for anyone browsing the repository. It explains how the code is
organised, how sample data flows from the input pins to the PC, and how the host controls the
capture.

## What it is

open-logic-bit is open-source logic analyzer gateware written in Verilog. It samples digital inputs
at up to 100MHz, run-length encodes them, streams them into external DDR3 memory, and lets a host PC
read the capture back over an FTDI USB sync-FIFO link. On the PC side it is used through
[sigrok / PulseView](https://sigrok.org/) via a
[libsigrok fork](https://github.com/ultraembedded/libsigrok) (host software is not in this repo).

The currently supported board is the
[Digilent Digital Discovery](../boards/digilent_digital_discovery/README.md) (Xilinx Spartan-6,
24 input channels, DDR3, FT232H USB).

![SPDIF capture](pulseview_spdif.png)

## Repository layout

```
boards/
  digilent_digital_discovery/
    top.v              Board top: pins, clock generation, MIG DDR3 instance, status LEDs
    fpga_top.v         System interconnect - instantiates and wires all core blocks
    fpga.ucf           Pin and timing constraints
    dcm12_48.v         12MHz -> 48MHz DCM (feeds the memory PLL)
    dcm12_100.v        12MHz -> 100MHz DCM (core + sample clock)
    spartan6_pll.v     48MHz -> 312MHz PLL (MIG system clock)
    reset_gen.v        Power-on reset generator (one per clock domain)
    prebuilt/fpga.bit  Ready-to-load bitstream
    ipcore_dir/mig/    Xilinx-generated DDR3 memory controller (treat as a black box)
core/
  capture/             The logic analyzer
    capture_rle.v          Input sampling, clock divider, test mode, RLE compression
    capture_rle_cdc.v      Async FIFO: sample clock -> core clock
    logic_capture.v        Register bank, trigger logic, sample FIFO
    logic_capture_defs.v   Register map definitions
    logic_capture_mem.v    Ring-buffer writer into memory (wraps logic_capture)
    logic_capture_mem_*.v  AXI master + helper FIFOs for the writer
    logic_capture_fifo.v   Sample FIFO
  fabric/              AXI4 plumbing
    axi4_arb.v             2:1 AXI4 arbiter (capture writer vs. host access)
    axi4_cdc.v             AXI4 clock-domain crossing (+ axi4_cdc_fifo*.v)
    axi4_lite_tap.v        Address decoder: registers vs. memory
    axi4_retime.v          Pipeline register slice
  ft245/               USB host link
    ft245_fifo.v           FTDI FT245 synchronous FIFO interface
    ft245_axi.v            Byte-stream command protocol -> AXI4 master
    ft245_axi_retime.v     Optional retiming of the AXI master port
docs/                  Screenshots and this document
```

## Block diagram

This is the structure of [`fpga_top.v`](../boards/digilent_digital_discovery/fpga_top.v) plus the
memory controller in [`top.v`](../boards/digilent_digital_discovery/top.v):

```
                 sample clk (100MHz)               core clk (100MHz)                       mem clk (~104MHz)
               ┌───────────────────┐   ┌──────────────────────────────────────────┐
 DIN[23:0] ───►│   capture_rle     │──►│  logic_capture_mem                       │
               │ resync, clk div,  │CDC│  ┌───────────────┐   ┌────────────────┐  │
               │ test counter, RLE │   │  │ logic_capture │──►│ ring-buffer    │  │
               └───────────────────┘   │  │ regs, trigger,│   │ AXI writer     │  │
                                       │  │ sample FIFO   │   │ (bursts)       │  │
                                       │  └───────▲───────┘   └───────┬────────┘  │
                                       └──────────┼───────────────────┼───────────┘
                                                  │ AXI-Lite          │ AXI4
                                                  │ (cfg regs)        ▼
                                                  │             axi4_retime
                                                  │                   │ port 0
                                                  │                   ▼
                                                  │             ┌──────────┐      ┌──────────┐     ┌─────────┐
                                                  │             │ axi4_arb │─────►│ axi4_cdc │────►│ MIG     │──► DDR3
                                                  │             └──────────┘      └──────────┘     │ (DDR3)  │
                                                  │                   ▲ port 1                     └─────────┘
                                                  │ addr ≥ 0x8000_0000│ addr < 0x8000_0000
                                             ┌────┴───────────────────┴────┐
                                             │       axi4_lite_tap         │
                                             └──────────────▲──────────────┘
                                                            │ AXI4
                                                       ┌────┴─────┐
                                                       │ axi4_cdc │
                                                       └────▲─────┘
                FTDI clk (60MHz)                            │ AXI4
               ┌────────────────────────────────────────────┴─┐
 USB / PC ◄───►│ ft245_axi  (ft245_fifo + command FSM)        │
  FT232H       └──────────────────────────────────────────────┘
```

Key points:

- The **host is an AXI master**. Every host operation (configure, poll status, read samples) is an
  AXI read or write issued by `ft245_axi`.
- `axi4_lite_tap` splits host traffic by address bit 31: `0x8000_0000` and above goes to the
  capture register bank, everything below goes to DDR.
- DDR is shared between the capture writer (arbiter port 0) and host reads (port 1) through
  `axi4_arb`.

## Clock domains

| Domain     | Frequency | Source                                         | Used by                                   |
|------------|-----------|------------------------------------------------|-------------------------------------------|
| FTDI       | 60MHz     | `FTDI_CLK` from the FT232H                     | `ft245_axi`, host side of `u_cdc`         |
| Core       | 100MHz    | `CLK12` → `dcm12_100`                          | Register bank, trigger, memory writer, fabric |
| Sample     | 100MHz    | Same as core on this board (`sample_clk_w`)    | Input capture / RLE in `capture_rle`      |
| MIG system | 312MHz    | `CLK12` → `dcm12_48` → `spartan6_pll`          | DDR3 controller internals                 |
| Memory     | ~104MHz   | MIG `c3_clk0`                                  | MIG AXI port, memory side of `u_cdc_mem`  |

Crossings happen in three places: `capture_rle_cdc` (sample → core), `u_cdc` (FTDI → core) and
`u_cdc_mem` (core → memory). Configuration signals that feed the sample domain are
double-registered in `capture_rle`.

## Dataflow walkthrough

### 1. Sampling and run-length encoding – [`capture_rle.v`](../core/capture/capture_rle.v)

- Inputs pass through a two-flop synchroniser.
- A clock divider (`CFG.clk_div`) sets how often a sample is taken.
- In **test mode** (`CFG.test_mode`) the inputs are replaced by an incrementing counter, which is
  handy for checking the whole pipeline without any external signals:

  ![Test mode counter](pulseview_counter.png)

- Each sample is compared with the previous one. A 32-bit word is emitted only when the input
  **changes** or the run counter **overflows**. Word format depends on `CFG.width`:

  | `CFG.width` | Channels | Output word                    |
  |-------------|----------|--------------------------------|
  | 0           | 16       | `{run_count[15:0], data[15:0]}` |
  | 1           | 24       | `{run_count[7:0],  data[23:0]}` |
  | 2           | 32       | `data[31:0]` (no compression)  |

- The words cross into the core clock domain through `capture_rle_cdc`.

### 2. Triggering and buffering – [`logic_capture.v`](../core/capture/logic_capture.v)

- Each channel's trigger is set by three registers:
  - `TRIG_ENABLE[i]` – channel takes part in the trigger
  - `TRIG_LEVEL[i]` – 1 = level-sensitive, 0 = edge-sensitive
  - `TRIG_SENSE[i]` – level: value to match; edge: 1 = rising, 0 = falling
- The trigger fires when **all** enabled channels match at the same time. If no channels are
  enabled, the capture starts as soon as it is enabled.
- For edge triggers, the sample just before the edge is also stored so the edge itself shows up in
  the capture.
- After the trigger, every RLE word goes into a sample FIFO. The FIFO separates sampling from
  memory stalls. If it overflows, `STS.data_loss` is set and the LED turns red.
- `SAMPLES` counts decoded samples (it adds up the RLE run counts).

### 3. Writing to memory – [`logic_capture_mem.v`](../core/capture/logic_capture_mem.v)

- Samples are written as 32-bit words into a ring buffer from `BASE` to `END`.
- When the write address is 32-byte aligned and at least 8 words are queued, the writer issues an
  8-beat AXI burst. Otherwise it writes single beats.
- **One-shot mode** (`CFG.cont = 0`): writing stops when `END` is reached.
  **Continuous mode** (`CFG.cont = 1`): the buffer wraps to `BASE` and `STS.wrapped` is set.
- `CURRENT` follows *completed* writes, so the host knows how much valid data is in memory.

### 4. Readback to the host

The host reads the capture straight out of DDR (addresses below `0x8000_0000`) with USB `READ`
commands. These go through `axi4_lite_tap` → `axi4_arb` port 1 → `u_cdc_mem` → MIG. The host then
decodes the RLE words back into samples.

## Register map

Defined in [`logic_capture_defs.v`](../core/capture/logic_capture_defs.v). Base address
`0x8000_0000`.

| Offset | Name          | Access | Fields                                                                                     |
|--------|---------------|--------|--------------------------------------------------------------------------------------------|
| `0x00` | `CFG`         | RW     | `[0]` enabled, `[1]` ext clock source, `[5:2]` clk_div, `[7:6]` width (0=16, 1=24, 2=32 ch), `[8]` test_mode, `[31]` continuous |
| `0x04` | `STS`         | RO     | `[0]` triggered, `[1]` wrapped, `[2]` data_loss, `[29:24]` num_channels                     |
| `0x08` | `BASE`        | RW     | Buffer start address in DDR (word aligned)                                                |
| `0x0C` | `END`         | RW     | Buffer end address in DDR (default `0x0200_0000` = 32MB)                                  |
| `0x10` | `CURRENT`     | RO     | Address reached by completed writes                                                       |
| `0x14` | `SAMPLES`     | RO     | Number of decoded samples captured                                                        |
| `0x18` | `TRIG_ENABLE` | RW     | Per-channel trigger enable mask                                                           |
| `0x1C` | `TRIG_SENSE`  | RW     | Per-channel level value / edge polarity (1 = rising)                                      |
| `0x20` | `TRIG_LEVEL`  | RW     | Per-channel mode: 1 = level, 0 = edge                                                     |

Setting `CFG.enabled` from 0 to 1 starts a new capture: the trigger, the sample count, the status
flags and the write pointer are all reset.

## USB host protocol – [`ft245_axi.v`](../core/ft245/ft245_axi.v)

The FTDI link carries **32-bit little-endian words**. Each command looks like this:

```
word 0 : { seq[31:16], len[15:8], cmd[7:0] }   header
word 1 : address
word 2+: data words (writes / echo only)
```

| `cmd`  | Name            | Behaviour                                                       |
|--------|-----------------|-----------------------------------------------------------------|
| `0x01` | ECHO            | Echoes `len` data words back, then a status word                |
| `0x02` | DRAIN           | Throws away incoming data until the RX FIFO is empty            |
| `0x10` | READ            | AXI burst read of `len` words; data is returned, then status    |
| `0x20` / `0x21` / `0x22` | WRITE8 / 16 / 32 | AXI write of `len` words (byte lanes chosen from the address); then status |
| `0x30` / `0x31` / `0x32` | WRITE8 / 16 / 32 (posted) | Same, but no status word is returned |
| `0x40` / `0x41` | GPIO WR / RD | Write / read the 8-bit GPIO port                              |

Status word: `{14'b0, axi_resp[17:16], seq[15:0]}`.

## Typical capture sequence (host side)

1. Write `CFG` with `enabled = 0` to stop any capture in progress.
2. Set up `TRIG_ENABLE`, `TRIG_SENSE` and `TRIG_LEVEL`.
3. Set the buffer window with `BASE` and `END`.
4. Write `CFG` with the width, clock divider, continuous/one-shot mode (and test mode if needed),
   with `enabled = 1`.
5. Poll `STS` and `CURRENT` until the trigger has fired and enough data has been written.
6. `READ` DDR from `BASE` up to `CURRENT` and decode the RLE words.

## Status LED

`LD1` (driven in [`top.v`](../boards/digilent_digital_discovery/top.v)):

| Colour | Meaning                     |
|--------|-----------------------------|
| Blue   | Capture enabled, waiting    |
| Green  | Triggered, capturing        |
| Red    | Sample FIFO overflow (data loss) |

## Building and loading

- A prebuilt bitstream is provided in
  [`boards/digilent_digital_discovery/prebuilt/fpga.bit`](../boards/digilent_digital_discovery/prebuilt/).
  Load it with `openFPGALoader -c digilent_ad prebuilt/fpga.bit`. See the
  [board README](../boards/digilent_digital_discovery/README.md) for details.
- The Digital Discovery has no flash for user bitstreams, so the gateware has to be reloaded after
  each power cycle.
- The design targets Xilinx Spartan-6 and is built with Xilinx ISE. The DDR3 controller is
  generated by the ISE MIG tool (`ipcore_dir/`).
- The repository does not include testbenches for the core.
