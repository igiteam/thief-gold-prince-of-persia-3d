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

The core question in that VOGONS thread—"what exactly are a3dapi.dll and a3d.dll, and why does A2D lack the latter"—can be answered clearly: a3dapi.dll is the A3D application programming interface (API) layer, and a3d.dll is the low-level hardware driver or compatibility layer for the specific sound card.

a3dapi.dll: The Stable API Layer
This is the library directly called by the game. It contains the complete A3D API implementation logic, responsible for managing sound sources, listener positions, and the software-side processing of the geometry engine (Wavetracing) .

What it does: The game tells a3dapi.dll "there's a sound source here," and it calculates the relevant parameters (such as occlusion, reflection, HRTF filtering) before sending instructions downward.

Why it's "stable": Regardless of what sound card you use, the game only recognizes this interface, so a3dapi.dll itself does not change with hardware.

a3d.dll: The Hardware/Driver Adaptation Layer
This is the low-level driver that "actually makes the sound card produce sound." It directly interfaces with the Aureal Vortex chip (or a third-party wrapper).

Why it "changes": Different sound cards (genuine Vortex, third-party cards using A3D-Live wrappers, software emulation) have completely different underlying implementations. a3d.dll acts as a "translator," forwarding the instructions processed by a3dapi.dll to the specific hardware or software renderer .

What it does: On genuine Vortex cards, it uses the chip's hardware acceleration; on non-Vortex cards, it may rely on software emulation (A2D) or a third-party wrapper to handle it.

Why Does A2D Lack a3d.dll?
A2D itself is a pure software emulation mode that does not require any dedicated hardware acceleration. In the A2D architecture, the "hardware driver adaptation layer" (a3d.dll) has no hardware to adapt to, so it is directly bypassed or built into the software processing flow of a3dapi.dll .

Moreover, A2D is not a full implementation of A3D. It only performs positioning on the horizontal plane and does not support HRTF, material filters, or real reflection calculations, so its sound quality is far inferior to genuine A3D . This is also why users in the thread said they "couldn't hear any reflections."

Additional Context: Can A3D Be Used on Modern Systems?
The most promising current approach is community reverse engineering. A project called re-a3d-api is reconstructing the A3D API and its compatibility library in C++, aiming to provide an A3D interface that can run on modern systems . Additionally, A3D-Live is a wrapper that can "trick" the game into thinking A3D hardware exists on modern Windows, but as users in the thread noted, it works "sometimes yes, sometimes no" and is unstable .

So, the two dlls in that old post you read work as a team: a3dapi.dll handles the "what to compute," and a3d.dll handles the "how to produce sound." A2D skips the hardware part and simply computes positioning in software, naturally requiring no dedicated a3d.dll.


Files in current category:

File Size Added Category ["Believe Your Ears" - Aureal A3D Demo Disc](getfile.php?fileid=1547&menustate=0) 312.85 MiB

2019-06-04  
21:52:44

Aureal [A2D Support Files v3.12](getfile.php?fileid=173&menustate=0) 227.79 KiB

2012-01-24  
20:49:52

Aureal [A3D Support Files v1.2.12 - 3.11](getfile.php?fileid=1940&menustate=0) 3.26 MiB

2022-07-18  
13:44:49

Aureal [A3D Support Files v3.12](getfile.php?fileid=174&menustate=0) 1.28 MiB

2012-01-24  
20:51:11

Aureal [AU8810 9x 4.06.3220 driver](getfile.php?fileid=343&menustate=0) 9.00 MiB

2012-03-16  
03:45:24

Aureal [AU8810 9x 4.06.3220 driver (full)](getfile.php?fileid=458&menustate=0) 35.08 MiB

2013-03-28  
20:11:07

Aureal [AU8820/AU8808 9x 4.06.1160 driver](getfile.php?fileid=454&menustate=0) 6.36 MiB

2013-03-28  
20:06:28

Aureal [AU8820/AU8808 9x 4.06.1185 driver](getfile.php?fileid=457&menustate=0) 6.52 MiB

2013-03-28  
20:06:28

Aureal [AU8820/AU8808 9x 4.06.1186 driver](getfile.php?fileid=455&menustate=0) 7.16 MiB

2013-03-28  
20:06:28

Aureal [AU8820/AU8808 9x 4.06.1187 driver](getfile.php?fileid=456&menustate=0) 9.25 MiB

2013-03-28  
20:06:28

Aureal [AU8820/AU8808 9x 4.06.1189 driver](getfile.php?fileid=175&menustate=0) 5.55 MiB

2012-01-24  
20:53:09

Aureal [AU8830 9x 4.06.2000 driver](getfile.php?fileid=453&menustate=0) 6.59 MiB

2013-03-28  
20:00:49

Aureal [AU8830 9x 4.06.2015 driver (recommended for DirectX &gt;= 7.1)](getfile.php?fileid=451&menustate=0) 21.70 MiB

2013-03-28  
20:00:49

Aureal [AU8830 9x 4.06.2031 driver](getfile.php?fileid=447&menustate=0) 7.63 MiB

2013-03-28  
20:00:49

Aureal [AU8830 9x 4.06.2035 driver](getfile.php?fileid=441&menustate=0) 33.93 MiB

2013-03-28  
20:00:49

Aureal [AU8830 9x 4.06.2040 driver](getfile.php?fileid=446&menustate=0) 34.55 MiB

2013-03-28  
20:00:49

Aureal [AU8830 9x 4.06.2040 INF Tweaks](getfile.php?fileid=443&menustate=0) 42.21 KiB

2013-03-28  
20:00:49

Aureal [AU8830 9x 4.06.2041 driver (recommended for DirectX &lt;=7.0a)](getfile.php?fileid=445&menustate=0) 34.53 MiB

2013-03-28  
20:00:49

Aureal [AU8830 9x 4.06.2041 INF Tweaks](getfile.php?fileid=448&menustate=0) 46.99 KiB

2013-03-28  
20:00:49

Aureal [AU8830 9x 4.06.2041 S/PDIF routing fix](getfile.php?fileid=1734&menustate=0) 7.71 KiB

2020-10-31  
16:07:52

Aureal [AU8830 9x 4.06.2048 driver](getfile.php?fileid=436&menustate=0) 36.85 MiB

2013-02-18  
03:49:43

Aureal [AU8830 9x 4.06.2048 EAX 2.06 Wrapper](getfile.php?fileid=442&menustate=0) 612.83 KiB

2013-03-28  
20:00:49

Aureal [AU8830 9x 4.06.2048 EAX 3.00 Wrapper](getfile.php?fileid=452&menustate=0) 87.75 KiB

2013-03-28  
20:00:49

Aureal [AU8830 9x 4.06.2048 HRTF Profiles](getfile.php?fileid=450&menustate=0) 248.04 KiB

2013-03-28  
20:00:49

Aureal [AU8830 9x 4.06.2048 INF Tweaks](getfile.php?fileid=449&menustate=0) 20.66 KiB

2013-03-28  
20:00:49

Aureal [AU8830 NT 4.0 2.05.05 driver](getfile.php?fileid=2266&menustate=0) 5.05 MiB

2025-08-11  
16:16:34

Aureal [Aureal Minerva Utility](getfile.php?fileid=1939&menustate=0) 4.83 MiB

2022-07-18  
13:44:49

Aureal [Diamond Monster Sound I, M80, MX200](getfile.php?fileid=417&menustate=0) 22.28 MiB

2012-12-27  
20:07:05

Aureal [Diamond Monster Sound II (MX300) Driver CD image](getfile.php?fileid=400&menustate=0) 59.52 MiB

2012-10-21  
07:36:08

Aureal [Diamond Monster Sound II (MX300) official downloads](getfile.php?fileid=416&menustate=0) 13.46 MiB

2012-12-27  
20:01:41

Aureal [Diamond Monster Sound II (MX300) Win9x NT4 v1.01 Beta](getfile.php?fileid=1943&menustate=0) 15.01 MiB

2022-07-18  
13:53:37

Aureal [Diamond Monster Sound II (MX300) Win9x NT4 v1.02](getfile.php?fileid=1941&menustate=0) 14.43 MiB

2022-07-18  
13:53:37

Aureal [Diamond Monster Sound M80 / MX200 Driver](getfile.php?fileid=1036&menustate=0) 7.60 MiB

2017-04-26  
22:41:30

Aureal [SQ2500 OEM Drivers](getfile.php?fileid=1025&menustate=0) 112.84 MiB

2017-04-26  
21:48:40

Aureal [Techworks Power Vortex2 Superquad Original Installer CD](getfile.php?fileid=399&menustate=0) 56.29 MiB

2012-09-17  
00:07:06

Aureal [Turtle Beach Montego II Applications CD Version 3.00](getfile.php?fileid=1009&menustate=0) 294.02 MiB

2016-12-26  
13:06:10

Aureal [Turtle Beach Montego II NT4 2.03](getfile.php?fileid=1944&menustate=0) 4.98 MiB

2022-07-18  
13:53:37

Aureal [Turtle Beach Montego II Win9x 2046](getfile.php?fileid=1942&menustate=0) 8.40 MiB

2022-07-18  
13:53:37

Aureal [Vortex Advantage / SoundCom Drivers (2001 from ASUS L8400 Laptop)](getfile.php?fileid=1031&menustate=0) 98.34 MiB

2017-04-26  
22:23:36

Aureal [Vortex2 SQ2500 Demo CD](getfile.php?fileid=843&menustate=0) 472.23 MiB

2015-11-09  
00:34:50

Aureal [Vortex2 SQ2500 Driver CD 2.040](getfile.php?fileid=846&menustate=0) 488.03 MiB

2015-11-09  
03:10:07

Aureal

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
