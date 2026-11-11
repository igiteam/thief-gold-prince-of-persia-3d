# Aureal A3D Software Development Kit
Aureal A3D is the landmark 3D audio technology which enables a realistic, natural and immerse audio experience by surrounding the listener with sounds in all three dimensions using only a single pair of ordinary speakers or headphones. Using advanced HRTF (head related transfer function) algorithms, A3D allows for interactive positioning of individual sound streams and listeners in true 3D space.  Application developers can easily utilize this powerful audio technology via the A3D API - a straightforward extension of the Microsoft DirectSound API.

![https://raw.githubusercontent.com/igiteam/aureal-a3d-sdk/refs/heads/main/Aureal_A3D_(logo).png](https://raw.githubusercontent.com/igiteam/aureal-a3d-sdk/refs/heads/main/Aureal_A3D_(logo).png)

A3D is the standard in PC system 3D sound, and is supported by many hardware platforms. This growing list of hardware products include Aureal’s own line of PC audio semiconductors - the Vortex PCI-based digital audio accelerator - as well as licensee products such as the Diamond MonsterSound, and Crystal SoundFusion.


# A3D SDK Notes & Integration Plan for `thief-gold-prince-of-persia-3d`

Notes on the archived Aureal A3D SDK and how an A3D-style audio backend would slot into the Thief Gold / Prince of Persia 3D remake pipeline.

---

## 1. What the SDK file confirms

**A3D is an extension of DS3D, not a standalone API.** The key line:

> "A3D is a superset of the DS/DS3D API – the added functionality is supported through extensions to DS/DS3D."

So any code written against A3D is really **DS3D + A3D extensions**. This matters for reimplementation: the base positional model is DirectSound3D, and A3D only adds the extras (wavetracing, occlusion/obstruction, environment params).

### Three SDK versions with dates

| Version | Date | Notes |
|---|---|---|
| A3D 1.2 | 1998/03 | HRTF positioning; `a3d.dll` + `A3D.vxd` drivers |
| A3D 2.0 | 1999/03/08 | Adds **Wavetracing** (the Vortex2 feature) |
| A3D 3.1 | 2000/02/11 | Adds Debug Viewer; I3DL2 preset support |

### Hardware

- Vortex (PCI) — Aureal's own chip
- Licensees: Diamond MonsterSound, Crystal SoundFusion

---

## 2. Useful artifacts

| Artifact | Use |
|---|---|
| **A3D 2.0/3.0 API Reference Guide** | Primary source for reimplementation: function list, structs, error codes |
| **A3D 2.0/3.0 User's Guide** | Integration model (listener, source, geometry, materials) |
| **Source code + sample apps** | Complete A3D code to read (call sequence, geometry hand-off) |
| **Wavetracing technology demo** | Ground truth for the effect (move sources, listener, walls) |
| **Minerva test application** | Behaviour reference for DS / DS3D / A3D compatibility |
| **Debug Viewer (3.0)** | *"visually display all the geometry, sources, reflections, and occlusions"* — shows what wavetracing computes |
| **Vortex datasheets** | Chip-level detail |

### Usable vs. historical

| Artifact | Usable for modern/WASM reimplementation? |
|---|---|
| API Reference Guide (2.0/3.0) | **Yes** — defines semantics: position, velocity, occlusion, obstruction, reverb params |
| User's Guide | **Yes** — integration model |
| Sample source code | **Yes, read-only** — call sequence and geometry hand-off |
| Wavetracing demo | **Yes as reference** — ground truth for behaviour |
| Debug Viewer | **Yes, conceptually** — which quantities matter (rays, reflections, occlusions) |
| Drivers (`a3d.dll`, `A3D.vxd`) | **No** — Win9x VxD, dead on modern Windows, irrelevant to WASM |
| Vortex datasheets | **Partly** — register detail, only needed for chip emulation |

---

## 3. Why the OpenAL mapping is clean

Because A3D is **DS3D + extensions**, the mapping to OpenAL is more direct than it looks:

| A3D concept | OpenAL equivalent | Notes |
|---|---|---|
| A3D **listener** | OpenAL listener (position, orientation, velocity) | Direct |
| A3D **source** | OpenAL source (position, velocity, cone, min/max distance) | Direct |
| A3D **occlusion / obstruction** | OpenAL filters (low-pass / direct-HF) | Driven by your geometry queries |
| A3D **environment / reverb** | OpenAL EFX reverb (I3DL2 family) | A3D 3.0 also supported I3DL2 |
| A3D **wavetracing** | **No OpenAL equivalent** | You compute rays yourself, then feed filters/EFX |

**Summary:** the SDK tells you *what the parameters are*; OpenAL gives you *where to put them*; the only thing written from scratch is the **geometry → parameter** step.

---

## 4. How this fits into `thief-gold-prince-of-persia-3d`

### 4.1 Repo layout (proposed)
thief-gold-prince-of-persia-3d/
├── engine/ # the source-port engine (Thief Gold / PoP3D)
├── audio/
│ ├── router/ # one interface, three backends
│ │ ├── router.h/.c # AudioRouter: play, set listener, set geometry
│ │ ├── backend_a3d.c # A3D-style: HRTF + geometry rays -> OpenAL filters
│ │ ├── backend_xbox.c # Xbox/Sensaura: OpenAL EFX presets (I3DL2)
│ │ └── backend_openal.c # native OpenAL HRTF + EFX
│ ├── geometry/ # level geometry -> audio polygons + materials
│ │ ├── trace.c # ray casting: occlusion, reflection, first/later
│ │ └── materials.c # material -> absorption/filter coefficients
│ └── dsp/ # HL2 DSP preset data (recipes, not drivers)
├── web/
│ ├── index.html # WASM shell
│ └── netlify.toml # wasm MIME, COOP/COEP only if pthreads
└── README.md

 
### 4.2 The router interface (what the game calls)

The game never talks to a specific backend. It calls the router:

```c
// audio/router/router.h
typedef enum { BACKEND_A3D, BACKEND_XBOX, BACKEND_OPENAL } AudioBackend;

void  Audio_Init(AudioBackend backend);
void  Audio_SetListener(pos, orient, velocity);
Sound Audio_PlaySource(sound_id, pos, velocity, min_dist, max_dist);
void  Audio_SetRoom(room_id, reverb_preset);   // Xbox/OpenAL path
void  Audio_SetGeometry(level_geometry);       // A3D path
void  Audio_Update(dt);                        // per frame: rays, filters
BACKEND_XBOX → Audio_SetRoom maps to OpenAL EFX reverb presets. Easy, faithful (same I3DL2 lineage).

BACKEND_OPENAL → native HRTF + EFX. Straightforward.

BACKEND_A3D → Audio_SetGeometry + Audio_Update run the geometry traces, then feed the results into OpenAL filters and EFX. This is the hard path.
```

4.3 The A3D backend in detail
per frame, per audible source:
  1. direct path          -> is it blocked? (occlusion)      -> low-pass filter
  2. partial obstruction  -> is it partially blocked?        -> direct-HF filter
  3. first reflection     -> cast ray, find wall hit         -> early reflection tap
  4. later reflections    -> a few rays or a precomputed IR  -> EFX reverb send
  5. material absorption  -> from materials.c                -> per-band attenuation
  => set OpenAL source filters + EFX send level
Thief Gold is the right first target: the original is the standout A3D2 example, and the source port retains the level geometry the traces need.

Prince of Persia 3D side: same router, different geometry source; the backend is unchanged.

4.4 The HL2 DSP presets
They are parameter recipes (Valve's reverb definitions), not an Xbox or A3D driver.

Use them to drive:

BACKEND_XBOX → map to the closest EFX reverb preset.

BACKEND_A3D / BACKEND_OPENAL → feed reverb parameters where geometry is absent or as a baseline.

Do not try to load them into A3D or Xbox hardware — the formats and hardware are different.

4.5 Build & host pipeline (WASM)
Build in a Codespace with a pinned Emscripten SDK:

```bash
git clone https://github.com/emscripten-core/emsdk.git
cd emsdk && ./emsdk install 3.1.x && ./emsdk activate 3.1.x
source ./emsdk_env.sh
```

Compile (engine + audio router):

```bash
emcc engine/... audio/router/... -o web/index.html \
  -s USE_SDL=2 -s USE_WEBGL2=1 -s FULL_ES3=1 \
  -O2 --preload-file assets
```

Extract artifacts — .wasm, .js/.mjs, .data, index.html — commit to web/ (or dist/) so they survive Codespace deletion.

Deploy to Netlify / Forgejo Pages with:

```toml
[[headers]]
  for = "/*.wasm"
  [headers.values]
    Content-Type = "application/wasm"
```

Add COOP/COEP only if the build uses pthreads.

4.6 Security / hosting rules (non-negotiable)
Codespaces = workshop, not storefront. Build there; host on static hosting.

Never leave a public Codespace port serving game data. It is both a copyright exposure and the exact abuse pattern documented in the Codespaces security research.

Engine code can be public; game data stays private. pak0.pk3 / doom.wad / Thief data out of public repos.

Review .devcontainer/ and .vscode/ in any repo you open a Codespace on. Auto-run tasks and lifecycle hooks execute code.

5. Next concrete step
The highest-value items from the archived SDK are:

A3D 2.0 API Reference Guide — specifically the occlusion/obstruction functions and the geometry/material structures.

Wavetracing demo — as reference behaviour.

Those two define exactly what the WASM/OpenAL backend has to reproduce. Once the API Reference is in the repo (or the occlusion/obstruction section is pasted), each A3D call can be mapped to its OpenAL/EFX equivalent, with the ones needing custom geometry code marked.

That turns the router interface from the abstract sketch above into a concrete function table.


## A3D 1.2 Software Development Kit
1998/03/??

The Aureal SDK was designed to be simple to use, given an understanding of DirectSound API. In particular, A3D is a superset of the DS/DS3D API – the added functionality is supported through extensions to DS/DS3D.  This means you can use Microsoft’s industry standard DirectSound3D API, and with minimal modification to your code, enable the extended features of A3D Interactive.

- A3D 1.2 API Documentation
- A3D drivers (a3d.dll, A3D.vxd)
- A3D logos
- Release notes
- Code support files (Including CA3DRenderer, CA3DListener, and CA3DSource classes and some more sample applications running A3D with source code available)

## A3D 2.0 Software Development Kit
1999/03/08

Welcome to A3D 2.0, the industry-leading 3D audio standard. The Aureal A3D 2.0 Software Development Kit (SDK) includes everything necessary to build a complete sound engine including 3D positional audio and other features into your next software application. Included in the kit are the following:

- A3D 2.0 API Reference Guide
- A3D 2.0 User’s Guide
- Source code and sample A3D 2.0 applications
- Code support files
- Release notes
- A3D drivers
- A3D logos
- Portable A3D demo
- Aureal Wavetracing technology demo
- Minerva test application
- Marketing presentation
- Vortex audio controller data sheets

## A3D 3.1 Software Development Kit
2000/02/11

Welcome to A3D 3.0, the industry-leading 3D audio standard. The Aureal A3D 3.0 Software Development Kit (SDK) includes everything necessary to build a complete sound engine including 3D positional audio and other features into your next software application. Included in the kit are the following:

- A3D 3.0 API Reference Guide
- A3D 3.0 User’s Guide
- A3D drivers
- A3D logos
- Source code and sample A3D 3.0 applications
- Debug Viewer
- Code support files
- Release notes
- Aureal Wavetracing technology demo
- Minerva test application
- Vortex audio controller data sheets

### The A3D API Reference Guide
This is the heart of the SDK. The A3D API Reference Guide contains a complete list of A3D functions, their descriptions, structures, and error codes. This is where you find the actual code that makes A3D work. As new features are added to A3D 3.0, the release notes will be updated and you will find the actual code in the reference manual. An index of A3D functions is included for easy reference.

### The A3D 3.0 User's Guide
This document contains all the information you need (aside from the actual API reference manual, which details specific code) to add support for A3D into your application. It has been separated from the actual A3D API reference for your convenience. 

### Source Code and Sample A3D 3.0 Applications
Also known as tutorials, we have included some very simple applications which utilize A3D 3.0, along with the source code used to compile them. This enables you to see what complete A3D code looks like up front. Starting from the sample code, you can make changes and recompile to experiment with settings and other parts of the code.

### Debug Viewer
A tool to visually display all the geometry, sources, reflections, and occlusions that are currently being rendered.

### Code Support Files
Included with the SDK are the necessary static library files (with source) and header files to compile with your application.

### Release Notes
This is where you will find the latest information regarding the status of the A3D SDK and its contents.

### A3D Drivers
Included with the SDK are the latest available drivers (debug and release) that A3D uses to interface with your sound card. These drivers are updated often and are also available on our website (www.a3d.com) for your convenience. To see exactly what the A3D code is doing at a given time, you can use the debug drivers to step through the code and receive valuable debug information.

### A3D Logos
Included with the SDK are various versions (MAC, PC, PhotoShop, Illustrator, etc.) of the A3D logo. Although A3D works on any audio hardware, it works best with A3D-enabled hardware. Let your customers know that you support their hardware to the fullest by displaying this logo on the box and on your web site. 

### Aureal Wavetracing Technology Demo
Optimally supported by Vortex2-based A3D cards, this demo works on any A3D sound card. It very clearly demonstrates the benefits of Aureal Wavetracing technology, a key feature of A3D Move sound sources, the listener, and walls around to dynamically affect the A3D experience.
(Available only on the SDK CD)

### Minerva Test Application
Minerva is a testing and profiling tool for audio drivers and cards. Minerva tests for compatibility with and features of DirectSound, DirectSound3D, and A3D.
(Available only on the SDK CD)

### Vortex Product Briefs
These documents detail the specifications and features of Aureal's popular PCI audio controllers and A3D accelerators, the Vortex chipset.
(Available only on the SDK CD)

https://sigmaco.org/aureal-a3d

## Archived and preserved by SIGMA

This software was taken as part of the history of computer games and is now archived and being preserved by the cybercultist museum of the Federação SIGMA.

Federação SIGMA, formerly doing business as SIGMA Co. (contraction of SIGMA Collective), is a cybercultist conglomerate, of public interest and private law, established to bring together, organize and orchestrate, in a single mass, all societies, service boards, operational columns, and their respective resources, assets and rights under the common authority and singular ownership of SIGMA, the Latin American science, technology and cyber engineering collective.

© 2011-2023 Federação SIGMA. All rights reserved; to their respective elaborators and collaborators.

Technical contact: culture@sigmaco.org.
