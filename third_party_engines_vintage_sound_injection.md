# Exploring 3rd-Party Software, Synth Engines & Vintage Sound Ingestion

> **Document Status:** Community Research & Technical Sandbox  
> **Target Devices:** Akai Sample Key / MPC Sample (Embedded Linux / Open-Source Firmware Layer)

---

## 1. Executive Summary

Online discussions and creator demonstrations around open-source hardware architectures (and their bridge to AI models like Claude) center on a recurring ambition: **turning compact, accessible hardware samplers into chameleon synths capable of housing legendary digital and analog instruments.**

Because devices in this category run on lightweight embedded ARM/Linux stacks, openness unlocks two primary avenues for sound injection:

1. **Host-Side Headless Synthesis (Direct Bridge):** Running low-overhead open-source DSP emulations on a host machine (or internal daemon) driven directly by Claude via OSC/MCP.
2. **Autonomous Multi-Sampling & Instrument Synthesis (Asset Ingestion):** Using AI agents to autonomously patch, multi-sample, truncate, loop, and compile `.xpm` keygroup instruments from classic vintage architectures (Yamaha DX7, Roland D-50, Fairlight CMI, Ensoniq Mirage, PPG Wave).

```
┌────────────────────────────────────────────────────────────────────────┐
│                        AI Sound Ingestion Engine                       │
│                                                                        │
│  [Prompt: "1984 Fairlight Arr1 Flute + DX7 E-Piano with chorus"]       │
│                                │                                       │
│                                ▼                                       │
│    ┌──────────────────────────────────────────────────────────────┐    │
│    │ Claude / MCP Engine Orchestrator                             │    │
│    └──────────────┬───────────────────────────────┬───────────────┘    │
│                   │                               │                    │
│    Path A: Headless Runtime Engine         Path B: Auto-Keygroup Prep   │
│                   ▼                               ▼                    │
│    ┌──────────────────────────────┐ ┌──────────────────────────────┐   │
│    │ TinySoundFont / CSound / Pure│ │ Multi-velocity WAV Slicing   │   │
│    │ Data / dexed-headless Engine │ │ + Root Key/Zone XML (.xpm)   │   │
│    └──────────────┬───────────────┘ └──────────────┬───────────────┘   │
└───────────────────┼────────────────────────────────┼───────────────────┘
                    ▼                                ▼
       ┌────────────────────────┐       ┌────────────────────────┐
       │   Direct Audio / OSC   │       │   MicroSD / USB Mass   │
       │   Stream over USB-C    │       │   Storage Deployment   │
       └───────────┬────────────┘       └────────────┬───────────┘
                   │                                 │
                   ▼                                 ▼
       ┌─────────────────────────────────────────────────────────┐
       │             Akai Sample Key / MPC Sample                │
       └─────────────────────────────────────────────────────────┘
```

---

## 2. Target Instrument Archetypes & Emulation Strategies

### A. The 80s Frequency Modulation (FM) Era: Yamaha DX7 / TX81Z
* **The Vintage Signature:** Crisp, glassy electric pianos, snappy digital slap basses, and complex metallic harmonics.
* **Open-Source Engine Candidate:** `dexed` (headless C++ engine) or `msfa` (Music Synthesizer for Android/Linux core).
* **AI Workflow:** 
  * Instruct Claude to decode or generate original 32-byte DX7 SysEx patch algorithms (Operator ratios, feedback paths, envelope decay stages).
  * Claude triggers headless rendering of key zones ($C1, F1, C2, F2 \dots C6$) and formats them into zero-latency velocity layers.

### B. Linear Arithmetic (LA) & Vector Synthesis: Roland D-50 & Prophet VS
* **The Vintage Signature:** Short PCM transient attack waves spliced into sustained subtractive resonant sustain stages.
* **Open-Source Engine Candidate:** SoundFont / SFZ players (`TinySoundFont`, `sfizz`) and granular samplers.
* **AI Workflow:**
  * Claude identifies and extracts vintage acoustic transients (chiff, pick, bow scrape) from public domain PCM libraries.
  * Claude layers these transients over analog-style saw/square waveforms generated in Python, automatically baking authentic 12-bit/32 kHz DAC jitter and coloration into the attack.

### C. The Early Sampler Crunch: Fairlight CMI Series IIx & E-mu SP-1200
* **The Vintage Signature:** Variable sample rates, anti-aliasing filter ringing, and classic 8-bit / 12-bit linear quantization noise.
* **Open-Source Engine Candidate:** SOX DSP processing scripts or custom Python bit-crusher pipelines.
* **AI Workflow:**
  * Claude runs automated audio transformation pipelines that downsample modern 24-bit audio to 22.05 kHz or 26.04 kHz, applying zero-order hold interpolation and non-linear companding curves to mimic vintage converters before copying files to device storage.

---

## 3. Method 1: The AI Autonomous Multi-Sampler Pipeline

Rather than manually sampling every key on a vintage synth, an MCP-connected agent can automate the entire multi-sampling and keygroup construction loop:

```
[Claude MCP Client]
       │
       ├──> 1. Sends MIDI Note-On ($C2, vel: 100) to headless VST/hardware synth
       ├──> 2. Records audio buffer (3 seconds sustain, 1.5 seconds release tail)
       ├──> 3. Detects fundamental frequency ($f_0$) to confirm root pitch
       ├──> 4. Truncates leading silence via RMS threshold
       ├──> 5. Writes sample to disk: /Samples/DX7_Tines/DX7_C2_v100.wav
       └──> 6. Compiles Akai-compatible Keygroup XML definition (.xpm)
```

### Reference Node.js Multi-Sampler Script (`auto_sample_worker.ts`)

```typescript
import { Client } from "@modelcontextprotocol/sdk/client/index.js";
import fs from "fs";
import path from "path";

interface KeygroupLayer {
  rootNote: number;
  lowNote: number;
  highNote: number;
  sampleName: string;
}

// Generate the Akai Program XML (.xpm) with key mapping
export function buildKeygroupXml(programName: string, layers: KeygroupLayer[]): string {
  const layerXml = layers
    .map(
      (l, idx) => `
    <Instrument number="${idx + 1}">
      <LowNote>${l.lowNote}</LowNote>
      <HighNote>${l.highNote}</HighNote>
      <RootNote>${l.rootNote}</RootNote>
      <SampleFile>${l.sampleName}</SampleFile>
      <FilterCutoff>127</FilterCutoff>
      <AmpEnvelope>
        <Attack>0.00</Attack>
        <Decay>1.20</Decay>
        <Sustain>0.80</Sustain>
        <Release>0.60</Release>
      </AmpEnvelope>
    </Instrument>`
    )
    .join("");

  return `<?xml version="1.0" encoding="UTF-8"?>
<MPCVObject>
  <Version>2.0</Version>
  <Program type="Keygroup">
    <ProgramName>${programName}</ProgramName>
    <Instruments>
      ${layerXml}
    </Instruments>
  </Program>
</MPCVObject>`;
}
```

---

## 4. Method 2: Embedded 3rd-Party Runtimes on Open Hardware

If open-source firmware enables direct userland shell access or custom package execution on the device SoC, lightweight runtime engines can be deployed directly onto the machine:

| Engine | Binary Footprint | Target Capability on MPC Sample / Key |
| :--- | :--- | :--- |
| **Pure Data (`pd-vanilla`)** | $< 10\text{ MB}$ | Modular FM synth patches, generative random sequencers, microtonal tuners. |
| **Faust (`faust2minimal`)** | Compiled C/C++ | Ultra-efficient virtual analog filters, tube overdrive emulation, physical modeling. |
| **TinySoundFont (`tsf`)** | Single C header ($< 100\text{ KB}$) | Instant playback of lightweight SoundFont (`.sf2`) banks from vintage keyboards. |
| **Fluidsynth** | $< 5\text{ MB}$ | General MIDI retro-soundsets (Roland Sound Canvas, Yamaha MU-series soundfonts). |

### Conversational Deployment via Claude

With an MCP server exposing device filesystem tools, a user prompt can look like this:

> **User Prompt:**  
> *"Claude, I want that atmospheric pad from the Blade Runner soundtrack (CS-80). Find or synthesize a dual-saw patch with slow pulse-width modulation and detune, sample it across 4 octaves, and install it onto Pad Bank C as an XPM instrument."*

**Agent Action Sequence:**
1. Generates the two detuned sawtooth waves with an LFO-modulated PWM model using Python's `scipy.signal`.
2. Applies a modeled $-24\text{ dB/oct}$ low-pass ladder filter sweep with gentle saturation.
3. Renders 8 key zones ($C1, G1, C2, G2, C3, G3, C4, G4$).
4. Formats and tags `.wav` chunk headers with root pitch metadata.
5. Emits `BladeRunner_CS80.xpm`.
6. Syncs the files via USB storage mode or transfers via an SCP/RNDIS network socket to `/media/sdcard/MPC Programs/`.

---

## 5. Curated Open-Source Sample & Instrument Sources

To fuel your AI-driven ingestion pipeline, these open repositories and specifications provide rich source material for vintage synthesis:

* **[Soundfont / SFZ Archives (Musical Artifacts)](https://musical-artifacts.com/):** Extensive community collection of Creative Commons SoundFonts (`.sf2`) covering Casio CZ, Roland Juno, Mellotron, and Oberheim units.
* **[Dexed Sysex Cartridge Repository](https://github.com/asb2m10/dexed):** Thousands of classic `.syx` patch banks compatible with algorithmic extraction.
* **[The Internet Archive Synth Collection:](https://archive.org/details/synth-samples)** Historic multi-sample libraries digitized from vintage sample CDs (Akai S1000/S3000 formatted collections).

---

## 6. Safety & Performance Best Practices

* **RAM Buffer Constraints:** Compact hardware (like the Sample Key or MPC Sample) often features conservative RAM configurations ($512\text{ MB} - 1\text{ GB}$). Configure your AI generator to cap keygroup patches at 16-bit / 44.1 kHz mono or stereo with a maximum of 4–6 key zones per program to prevent out-of-memory crashes.
* **Click & Pop Suppression:** Always ensure your automated audio generation scripts apply a $2\text{ ms} - 5\text{ ms}$ linear fade-in and equal-power fade-out at sample boundaries to prevent harsh DC offsets and speaker pops.
* **Volume Staging:** Normalize all synthesized assets to $-3.0\text{ dBFS}$ peak true ceiling prior to transferring to the sampler to avoid internal clipping when applying onboard hardware saturation or distortion.
