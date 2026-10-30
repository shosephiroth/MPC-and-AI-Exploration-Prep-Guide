# MPC & AI Exploration: Comprehensive Field Guide & Index

> **Welcome to the Central Index.**  
> This document serves as the primary master guide, directory, and conceptual roadmap for interfacing open-source AI workflows, Anthropic's Model Context Protocol (MCP), and Open Sound Control (OSC) with modern Akai hardware (including the MPC Sample, Sample Key, and MPC Live III).

---

## 🧭 Guide Table of Contents & Module Index

Use this directory to jump directly into specific technical tracks and sub-guides:

| Module | Topic | Core Technologies | Status |
| :--- | :--- | :--- | :--- |
| **[Module 01](#module-01-local-mcp-servers-for-midi--hardware-control)** | Local MCP Server Setup | Node.js, TypeScript, Claude Desktop, Virtual MIDI | Planned / In Draft |
| **[Module 02](#module-02-hardware-architecture--tier-comparison)** | Hardware Architecture Comparison | MPC Sample / Sample Key vs. MPC Live III | Planned / In Draft |
| **[Module 03](#module-03-open-sound-control-osc-for-hardware-control)** | OSC Fundamentals for Grooveboxes | UDP Sockets, OSC Address Schemas, Floating-point control | Planned / In Draft |
| **[Module 04](#module-04-programmatic-sample-chopping--xml-generation)** | Programmatic `.xpm` & File Synthesis | Python, XML ASTs, transient slicing (`librosa`) | Planned / In Draft |
| **[Module 05](#module-05-micro-timing-swing-injection--generative-patterns)** | Generative Sequencer & Micro-Timings | Millisecond offsets, humanized velocity curves | Planned / In Draft |
| **[Module 06](#module-06-usb-gadget-networking-rndis--cdc-ecm)** | Direct Hardware Networking | Linux USB Gadget, Ethernet over USB-C | Planned / In Draft |

---

## Module 01: Local MCP Servers for MIDI & Hardware Control

The **Model Context Protocol (MCP)** by Anthropic allows Large Language Models (LLMs) such as Claude to invoke local tools through structured JSON schemas. Rather than generating raw shell scripts, the AI triggers deterministic functions like `trigger_pad_note` or `send_midi_cc`.

### Architectural Data Flow

```
┌─────────────────┐
│ Claude Desktop  │
└────────┬────────┘
         │ (stdio / JSON-RPC)
         ▼
┌─────────────────────────────────┐
│ Local Node.js MCP Server        │
│ (translates schema to MIDI/OSC) │
└────────┬────────────────────────┘
         │ (Virtual MIDI / UDP Socket)
         ▼
┌─────────────────────────────────┐
│ OS Virtual MIDI Bus / USB-C     │
│ (IAC Driver / loopMIDI / ALSA)  │
└────────┬────────────────────────┘
         │
         ▼
┌─────────────────────────────────┐
│ MPC Hardware (Sample / Live)    │
└─────────────────────────────────┘
```

### Quick-Start Setup

1. **Initialize Workspace:**
   ```bash
   mkdir mpc-mcp-bridge && cd mpc-mcp-bridge
   npm init -y
   npm install @modelcontextprotocol/sdk easymidi osc dotenv
   npm install -D typescript @types/node tsx
   ```

2. **Establish Loopback Ports:**
   * **macOS:** Audio MIDI Setup → Window → MIDI Studio → IAC Driver → Add port `Claude_To_MPC`.
   * **Windows:** Install [loopMIDI](https://www.tobias-erichsen.de/software/loopmidi.html) and launch a port named `Claude_To_MPC`.
   * **Linux:** Initialize `snd-virmidi` via ALSA.

3. **Claude Desktop Integration:**
   Register your script in your `claude_desktop_config.json`:
   ```json
   {
     "mcpServers": {
       "mpc_bridge": {
         "command": "npx",
         "args": ["-y", "tsx", "/ABSOLUTE/PATH/TO/mpc-mcp-bridge/index.ts"]
       }
     }
   }
   ```

---

## Module 02: Hardware Architecture & Tier Comparison

Understanding the resource constraints and intended workflow of your target device dictates how AI tasks should be architected.

```
┌──────────────────────────────────────┐     ┌──────────────────────────────────────┐
│       MPC Sample / Sample Key        │     │             MPC Live III             │
├──────────────────────────────────────┤     ├──────────────────────────────────────┤
│ • Embedded low-power SoC             │     │ • Multicore high-performance CPU     │
│ • Minimal display / hardware knobs   │     │ • 7" Multi-touch color display       │
│ • Lean embedded Linux OS             │     │ • Full standalone MPC3 OS            │
│ • MicroSD / Flash storage            │     │ • 8 GB RAM, 128 GB SSD + SATA bay    │
│ • Ideal as: Interactive Sound Module │     │ • Ideal as: Standalone Host / Studio │
└──────────────────────────────────────┘     └──────────────────────────────────────┘
```

### Detailed Feature Matrix

| Capability | MPC Sample / Sample Key | MPC Live III | Workflow Implication |
| :--- | :--- | :--- | :--- |
| **Form Factor** | Compact, battery/bus-powered, 16 mini pads or 25-key bed | Full workstation console, standard 16 velocity/aftertouch pads | **Sample:** Fast portable sketching.<br>**Live III:** Deep production & mixing. |
| **Processing Power** | Low-overhead DSP/ARM SoC | High-performance 8-core CPU | **Sample:** Offload AI processing to PC.<br>**Live III:** Potential for onboard local model inference. |
| **I/O Connectivity** | USB-C (Class-compliant MIDI/Audio), 3.5mm I/O | Dual XLR/TRS combos, 6 line outs, Phono, 8 CV/Gate, Ethernet | **Sample:** Streamlined digital control.<br>**Live III:** Multi-bus routing and CV modular sequencing. |
| **Primary AI Role** | Real-time parameter tweaking, patch design, remote slice triggering | Automated gain staging, dynamic arrangement, full project generation | Determines tool complexity in your MCP server. |

---

## Module 03: Open Sound Control (OSC) for Hardware Control

While MIDI 1.0 is historically constrained to 7-bit resolution (`0`–`127`) and raw hex bytes, **Open Sound Control (OSC)** uses dynamic, human-readable URL addresses and 32-bit floating-point precision over UDP networks.

### Anatomy of an OSC Message

An OSC packet consists of an **Address Pattern**, a **Type Tag**, and **Arguments**:

```
/track/01/filter/cutoff   ,f   0.725
└─── Address Pattern ───┘  │   └── 32-bit Float value (0.0 to 1.0)
                           └────── Type Tag indicator
```

### Common OSC Address Conventions for Samplers

* `/transport/play`, `/transport/stop`, `/transport/bpm`
* `/pad/{1-16}/trigger` (with velocity parameter)
* `/pad/{1-16}/pitch/tune` (semitone or fine cent float)
* `/mixer/ch/{1-16}/volume`, `/mixer/ch/{1-16}/pan`
* `/fx/{slot}/param/{id}`

### Why OSC Excels with AI & LLMs

1. **Human & Machine Legibility:** Large language models predict clear tokens like `/filter/cutoff` with near-zero hallucination compared to raw SysEx strings or NRPN byte pairs.
2. **Deterministic Tool Schema Matching:** Each address path directly translates to an MCP JSON tool declaration, enabling automatic type validation before data reaches your gear.

---

## Module 04: Programmatic Sample Chopping & XML Generation
*(Detailed walkthrough upcoming in separate module file)*

* **Concept:** Bypassing manual hardware slicing by using Python audio libraries (`librosa`, `scipy`) to detect onset transients in audio files.
* **Akai File Structure:** Automating the generation of `.xpm` (XML Program Maps) and `.xpj` project directories directly to MicroSD.
* **AI Integration:** Asking Claude to evaluate transient dynamics, label slices by sonic character (kick, rim, ghost note), and map them logically across pad banks A–D.

---

## Module 05: Micro-Timing, Swing Injection & Generative Patterns
*(Detailed walkthrough upcoming in separate module file)*

* **Beyond Static Swing:** Generating humanized groove templates that introduce micro-delays (e.g., pulling snare hits late by 10–25 ms while pushing hats ahead).
* **Conversational Groove Directives:** Prompting Claude to generate complex polyrhythms and Euclidean distributions, outputting them straight to standard MIDI files or live step-sequencer memory.

---

## Module 06: USB Gadget Networking (RNDIS / CDC-ECM)
*(Detailed walkthrough upcoming in separate module file)*

* **Ethernet-over-USB:** Leveraging embedded Linux USB Gadget drivers to create a high-speed, point-to-point network connection over a single USB-C cable.
* **Socket Communication:** Running bidirectional OSC/UDP daemons directly between the host computer and the MPC filesystem without requiring Wi-Fi access points.

---

## 🛠️ Step-by-Step Exploration Checklist

- [ ] **Hardware Setup:** Connect your MPC device to your development host via USB-C.
- [ ] **MIDI Loopback:** Verify active virtual ports using an OS monitor (e.g., *MIDI Monitor* on macOS, *MIDI-OX* on Windows).
- [ ] **Network / OSC Monitor:** Install an inspection tool like [Hexler Protokol](https://hexler.net/protokol) to monitor test UDP packets on chosen ports (e.g., `8000` / `9000`).
- [ ] **MCP Server Runtime:** Spin up your local bridge server (`npm run dev` or `tsx index.ts`).
- [ ] **Claude Desktop Verification:** Confirm that the MCP bridge tools appear under Claude's attachment/tools icon.
- [ ] **End-to-End Test:** Run the initial verification prompt:
  > *"Send a MIDI note trigger to Pad 1 at velocity 110, then set CC 74 to value 64."*

---

## 📚 References & Upstream Projects

* [Model Context Protocol Documentation](https://modelcontextprotocol.io/) — Official Anthropic MCP specification.
* [anteriovieira/osc-mcp-server](https://github.com/anteriovieira/osc-mcp-server) — Inspiration for OSC-to-MCP translation layers.
* [The Open Sound Control 1.0 Specification](https://opensoundcontrol.stanford.edu/spec-1_0.html) — Technical standard reference.
* [Project Readme & Community Disclaimer](./README.md) — Operational guidelines and experimental advisory.
