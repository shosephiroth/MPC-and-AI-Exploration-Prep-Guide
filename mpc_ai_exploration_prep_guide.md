# Field Prep Guide: MPC Ecosystem & AI Agent Integration

This technical guide prepares your studio, software environment, and mental model for interfacing Large Language Models (LLMs) via the **Model Context Protocol (MCP)** and **Open Sound Control (OSC)** with modern Akai hardware—specifically the lightweight **MPC Sample / Sample Key** architecture and the flagship **MPC Live III**.

---

## 1. Setting Up a Local MCP Server for MIDI & OSC Control

The **Model Context Protocol (MCP)** by Anthropic allows Claude Desktop (or local agent runners) to call tools registered in JSON schema formats. Instead of executing arbitrary shell commands, Claude invokes specific musical methods like `set_fader`, `trigger_pad`, or `send_cc`.

### Architecture Overview

```
[ Claude Desktop ]
       │  (stdio / JSON-RPC)
       ▼
[ Local Node.js MCP Server ]
       │  (UDP Sockets / WebMIDI / easymidi)
       ▼
[ Virtual MIDI Bus or OSC Network Socket ]
       │  (USB-C Cable / Wi-Fi Network)
       ▼
[ MPC Hardware (MPC Sample / Live III) ]
```

### Step-by-Step Configuration

#### Step 1: Install Dependencies
Ensure you have Node.js (v18+) and Claude Desktop installed.

```bash
# In your development/audio scripts folder:
mkdir mpc-mcp-bridge && cd mpc-mcp-bridge
npm init -y
npm install @modelcontextprotocol/sdk easymidi osc dotenv
npm install -D typescript @types/node tsx
```

#### Step 2: Establish Virtual MIDI Ports
* **macOS:** Open **Audio MIDI Setup** > Window > Show MIDI Studio > double-click **IAC Driver** > check **Device is online** > create a bus called `Claude_To_MPC`.
* **Windows:** Install **loopMIDI** by Tobias Erichsen and create a port named `Claude_To_MPC`.
* **Linux:** Use `snd-virmidi` or ALSA/JACK virtual patchbays.

#### Step 3: Minimal MCP MIDI Bridge Server
Create an entry point `index.ts`:

```typescript
import { Server } from "@modelcontextprotocol/sdk/server/index.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import {
  CallToolRequestSchema,
  ListToolsRequestSchema,
} from "@modelcontextprotocol/sdk/types.js";
import easymidi from "easymidi";

// Connect to virtual port routed to your MPC
const midiOut = new easymidi.Output("Claude_To_MPC", false);

const server = new Server(
  { name: "mpc-midi-bridge", version: "1.0.0" },
  { capabilities: { tools: {} } }
);

server.setRequestHandler(ListToolsRequestSchema, async () => ({
  tools: [
    {
      name: "send_midi_cc",
      description: "Send a MIDI Control Change message to control MPC parameters (filters, volume, pans)",
      inputSchema: {
        type: "object",
        properties: {
          channel: { type: "integer", minimum: 1, maximum: 16 },
          controller: { type: "integer", minimum: 0, maximum: 127 },
          value: { type: "integer", minimum: 0, maximum: 127 },
        },
        required: ["channel", "controller", "value"],
      },
    },
    {
      name: "trigger_pad_note",
      description: "Trigger a drum pad or melodic note on the MPC",
      inputSchema: {
        type: "object",
        properties: {
          channel: { type: "integer", minimum: 1, maximum: 16 },
          note: { type: "integer", minimum: 0, maximum: 127, description: "Pad 1 is typically Note 36 (C1)" },
          velocity: { type: "integer", minimum: 1, maximum: 127 },
          duration_ms: { type: "integer", default: 150 },
        },
        required: ["channel", "note", "velocity"],
      },
    },
  ],
}));

server.setRequestHandler(CallToolRequestSchema, async (request) => {
  if (request.params.name === "send_midi_cc") {
    const { channel, controller, value } = request.params.arguments as any;
    midiOut.send("cc", { controller, value, channel: channel - 1 });
    return { content: [{ type: "text", text: `Sent CC ${controller} -> Val ${value} on Ch ${channel}` }] };
  }

  if (request.params.name === "trigger_pad_note") {
    const { channel, note, velocity, duration_ms = 150 } = request.params.arguments as any;
    midiOut.send("noteon", { note, velocity, channel: channel - 1 });
    setTimeout(() => {
      midiOut.send("noteoff", { note, velocity: 0, channel: channel - 1 });
    }, duration_ms);
    return { content: [{ type: "text", text: `Triggered Note ${note} (Vel ${velocity}) on Ch ${channel}` }] };
  }

  throw new Error("Unknown tool");
});

const transport = new StdioServerTransport();
await server.connect(transport);
```

#### Step 4: Register with Claude Desktop
Edit your `claude_desktop_config.json`:
* **macOS:** `~/Library/Application Support/Claude/claude_desktop_config.json`
* **Windows:** `%APPDATA%\Claude\claude_desktop_config.json`

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

Restart Claude Desktop. You will see a hammer icon indicating Claude can now fire MIDI events directly to your device.

---

## 2. Hardware Comparison: MPC Sample vs. MPC Live III

Understanding the distinct hardware tiers determines which AI tasks make sense on each device:

| Feature / Metric | MPC Sample / Sample Key | MPC Live III |
| :--- | :--- | :--- |
| **Role & Philosophy** | Portable, fast, entry-level/hacker friendly | Flagship standalone mobile workstation |
| **Form Factor** | Compact, minimal controls (or 25-key mini keybed) | Full console: 16 MPCe 3D pads, touch strip, step keys |
| **Display** | High-contrast small OLED / 2-digit LED | 7" Multi-touch color display |
| **Internal OS / Engine** | Lightweight embedded Linux (Open-source accessible) | MPC3 OS (Custom multicore standalone build) |
| **RAM & Storage** | Lightweight flash memory + MicroSD | 8 GB RAM, 128 GB internal SSD + SATA drive bay |
| **Processing Power** | Embedded low-power SoC (DSP / basic ARM) | High-performance 8-core CPU |
| **Connectivity** | USB-C (Power, MIDI, Audio/Data), 3.5mm I/O | Dual XLR/TRS combos, 6 outputs, Phono, 8 CV/Gate, Ethernet/Wi-Fi |
| **Target AI Workflow** | **Remote agent target:** Claude treats it like an outboard sound generator, step sequencer, and transient slicer. | **Hybrid power station:** Onboard stems + offline model host; Claude scripts entire multi-track compositions, mixdowns, and patch routings. |

---

## 3. Open Sound Control (OSC) Basics for Music Gear

While traditional MIDI 1.0 is limited to 7-bit resolution (0–127) and cryptic status bytes, **OSC (Open Sound Control)** uses human-readable URL-style paths and 32-bit floating-point precision over UDP networks.

### Core Concepts

1. **Address Patterns:** Hierarchical strings resembling file paths:
   * `/channel/1/fader`
   * `/pad/4/filter/cutoff`
   * `/master/limiter/ceiling`
2. **Type Tags & Arguments:** Every argument has an explicit type:
   * `f` (32-bit Float, usually normalized `0.0` to `1.0`)
   * `i` (32-bit Integer)
   * `s` (String, e.g., sample names or preset labels)
   * `b` (Blob, used for transferring raw sysex or mini audio chunks)
3. **Bundles and Timetags:** OSC supports packaging multiple instructions together with microsecond timestamps (`#bundle`), ensuring simultaneous pad triggering without jitter.

### Why OSC is the Ideal Bridge for LLMs
As demonstrated in the `osc-mcp-server` repository (originally mapped to Behringer X32 OSC schemes):
* An LLM understands natural strings like `/ch/01/mix/fader 0.75` much more reliably than memorizing hex-based SysEx dumps or non-standard NRPN pairs.
* OSC schema maps 1:1 with JSON tool calling schemas, letting the model validate arguments before dispatching them.

---

## 4. Additional High-Value Workflows to Explore

### A. Programmatic Sample Chopping & `.xpm` Generation
Do not wait for real-time firmware hacks to start integrating AI into your MPC. Akai drum programs (`.xpm`) and sequence files (`.mid` / `.xpj`) are XML-based formats:
* **The Workflow:** Write a Python/Node.js script that uses `pydub` or `librosa` to analyze a `.wav` file, compute transient slice markers, and ask Claude to generate an `.xpm` program file assigning each transient slice to Pads 1–16 with tuned pitch offsets.
* Put the output directly onto the MicroSD card and plug it straight into the MPC Sample.

### B. Micro-timing and Swing Pattern Injectors
* Traditional groove templates only offer static swing percentages.
* With an MCP sequencer tool, you can instruct Claude:
  > *"Generate a 2-bar boombap hi-hat pattern on Pad 8 at 88 BPM, pushing off-beat 16ths late by 12–18 milliseconds, and varying velocity between 64 and 98 to mimic a human drummer playing a loose funk groove."*
* Claude computes the exact millisecond offsets and writes them via MIDI CC/Notes or standard Type-0 MIDI files.

### C. USB Gadget Mode (RNDIS / CDC-ECM) Networking
* Embedded Linux hardware platforms frequently support USB Gadget mode.
* When connected to your PC or Mac via USB-C, the MPC can present itself as a virtual network adapter (Ethernet over USB).
* This allows your local MCP server to communicate with the MPC via zero-latency UDP/OSC packets without requiring an external Wi-Fi router or physical MIDI cables.

---

## 5. Practical Studio Checklist

- [ ] **Hardware Setup:** Connect MPC Sample or Live III to computer via USB-C.
- [ ] **Audio/MIDI Loopback:** Verify virtual MIDI port is active in your DAW or OS utility.
- [ ] **Test Utility:** Install an OSC monitor (such as *Protokol* by Hexler) to inspect incoming/outgoing packets.
- [ ] **MCP Server Runtime:** Clone or build a simple MCP wrapper (Node.js or Python) exposing `easymidi` or `node-osc`.
- [ ] **Claude Desktop Config:** Add the bridge to `claude_desktop_config.json` and verify the hammer tools icon appears.
- [ ] **First Prompt Test:** Ask Claude: *"Send a test note to Pad 1 at velocity 100, then adjust CC 74 (Filter Cutoff) to 64."*
