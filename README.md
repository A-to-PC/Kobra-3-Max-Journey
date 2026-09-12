# Anycubic Kobra 3 Max: The Long Way Round

*A real ownership log — twelve days from unboxing to a fully custom, LAN-only monitoring and calibration setup: the mesh-repair bug that broke every calibration wizard, two real firmware lockups, a decrypted `.swu`, and three original apps built to fill the gaps stock firmware leaves open.*

This is a journey log, not an install or calibration manual — read it for what actually happened and why, not as a step-by-step to follow blind.

> new k3m, rinkhals installed, printer ran but hard to callibrate, went back to stock and slicer next, printer ran heaps quieter but can't monitor, build a webui, most of the slicer next cal prints have manifold issue, repair and run prints with no cal settings after repair, get frustrated try building a cal tool, play some more with slicer next and worked out not to repair after trying orca 3mf and gcode, gcode crashed the printer, it has been a hell of time to callibrate and get clean prints
>
> — *the whole story, in one breath, as I described it to Claude at 6:30am after the calibration marathon that finally cracked it*

This is the full log, not the highlight reel. If you've just bought a Kobra 3 Max, the short version is: it's a good printer, but the calibration wizards in Anycubic's own slicer are quietly broken in a way that produces flat, useless results if non-manifold errors are repaired, and stock firmware gives you no way to watch a print from anywhere except standing in front of it if you don't want to use their cloud. Everything below is what actually happened figuring both of those out, plus this printer's own real, tested calibration numbers (a starting point, not a universal answer — see [Chapter 5](#chapter-5-my-confirmed-calibration)) and the tools that came out of it.

> **A note on how this got built.** I've got 25 years in IT, and yes, I did and do use AI to help build the tools in this guide. No apology for that — AI can write in minutes what would genuinely take me months, and every technical claim below was verified against real hardware before it went in, not taken on faith from a chatbot. The judgment calls, the debugging instincts, and every hour actually spent staring at a printer were mine. The typing speed just isn't the point people think it is.

## The 12 Days

Skip to any chapter below for the full depth. This is just the shape of it.

| Day | | |
|---|---|---|
| **D1** | **Arrival & Rinkhals** | Kobra 3 Max arrives, picked up on clearance sale pricing. Rinkhals installed for root access and real monitoring — immediately flashed the wrong package variant and it self-removed on reboot. |
| **D2-6** | **Living with Rinkhals** | Correct package reflashed and confirmed persistent — real root access, real Klipper/Moonraker, genuinely working. OrcaSlicer's "Upload and Print" needed a workaround (upload only, start manually) — a real extra step every print, and combined with the printer running quieter on stock firmware, that's what tipped the scale back. |
| **D6** | **Back to stock, for now** | Printer ran noticeably quieter on stock firmware. Between that and the week's rough edges, went back to factory firmware + Anycubic Slicer Next — Rinkhals stays on the list to revisit later. |
| **D7** | **No monitoring on stock → Kobra LAN Monitor** | Stock firmware's remote monitoring is cloud-or-nothing: skip Anycubic's phone app and cloud account and run LAN mode instead, and you get zero visibility into the printer. Reverse-engineered the real local LAN/MQTT protocol instead and built a proper web dashboard — no cloud account, no Anycubic app, nothing hardcoded. |
| **D11** | **The calibration wizards break** | Slicer Next's temperature/retraction/max-flow wizards all print a single flat value instead of a real sweep — no error, no warning. Chased it for hours, got frustrated enough to start building an independent calibration tool from scratch. |
| **D11** | **The real bug, and two real lockups** | Found the actual cause (mesh repair silently orphans calibration settings) while, in parallel, testing OrcaSlicer as an alternative — which hard-locked the printer twice, later traced to a command-flooding bug in its Klipper output. |
| **D11-12** | **Firmware archaeology** | Decrypted the real `.swu` firmware package (publicly known password, zero risk), read the actual Go server binaries, and got ground-truth answers no documentation anywhere provides — real safety limits, the real upload pipeline, the real USB update trigger. |
| **D12** | **Clean prints, for real** | Full calibration confirmed against real hardware — temperature, flow, pressure advance, retraction, max volumetric speed, all three speed modes. Kobra LAN Monitor packaged into a proper installer and released. A second print-quality mystery (blobby embossed text) traced to a slicer wall-order setting, not the calibration at all. |

---

## Chapter 1: Trying Rinkhals First

> **TL;DR** — For anyone on the fence about the risk of trying Rinkhals: this is a real account of what it's actually like to run, not a warning-off. It's a genuinely impressive project — real root access, real Klipper/Moonraker running underneath Anycubic's own UI, on a printer nobody expected to get either. It's also under active, fast-moving development, and a handful of rough edges at that stage of its life made this particular week better spent going back to stock. Reinstalling it later, once things settle, is very much still the plan.

The Kobra 3 Max shipped with an Anycubic ACE Pro (the 4-slot multi-material unit) and no meaningful way to monitor a print remotely on stock firmware. [Rinkhals](https://github.com/rinkhals-community/Rinkhals) was the obvious fix — a community firmware layer that adds real Klipper/Moonraker underneath Anycubic's own UI, without fully replacing it. That's not a small thing to pull off on a closed platform like this, and it works.

> **First thing to get right: two different .swu files.** Anycubic's `.swu` firmware packages aren't one-size-fits-all, and Rinkhals ships more than one variant. An 8MB `installer-*.swu` is an interactive management tool, not Rinkhals itself — flashing that alone gets removed on the next power cycle. The real, persistent package is the 71MB `update-*.swu`. Easy mix-up, cost a full reinstall the first time.

Once the correct package was on, it held — survived a power cycle, then self-updated to a newer release and survived that too. Real, working root access, genuinely delivered.

### The thing that actually mattered: "Upload and Print"

Every attempt to upload and start a print in one step from OrcaSlicer failed with Anycubic error 10115, "The device cannot parse the file" — which reads like a gcode problem. It wasn't. Clicking through the error dialog (not just reading the summary line) surfaced the real traceback: Rinkhals' own Moonraker component (`kobra.py`, `handle_gcode_print_file`) delegates the auto-start trigger to the native firmware's print-start command, and *that* was what rejected it. Worth flagging honestly: LAN mode may not have been switched on at the time, which is a plausible alternate explanation for the native command being rejected — not confirmed as a pure Rinkhals automation bug independent of that, and never cross-checked against Rinkhals' own issue tracker.

> **The workaround, and why it wasn't enough to stay.** Click **Upload** only, never "Upload and Print." The upload itself always completed fine over Moonraker; only the automatic start trigger was affected. Start the print manually from the touchscreen or the app afterward — a workable fix, but one extra manual step on every single print adds up fast, and combined with the printer running noticeably quieter on stock firmware anyway, it wasn't worth staying on for the week this happened.

Rinkhals went back on the shelf for now — genuinely worth what it offers, and worth reinstalling once things settle a bit more, but this particular week called for print time over firmware debugging.

One honest caveat worth stating plainly, for anyone weighing the risk of trying Rinkhals themselves: the calibration marathon documented in the rest of this guide happened on stock firmware, not because stock is inherently easier to calibrate. It's simply where the time got spent. Put the same hours into calibrating under Rinkhals that went into stock here, and it would very probably have gotten sorted there too.

---

## Chapter 2: Back to Stock, and Building the Monitor Nobody Shipped

> **TL;DR** — Stock Anycubic firmware's only remote monitoring is its own phone app and cloud account — turn on LAN mode instead and you get nothing back, not even a local web page. So the local LAN protocol got reverse-engineered — with two existing community repos found and used along the way to cross-check the findings — and [Kobra LAN Monitor](https://github.com/A-to-PC/kobra-lan-monitor) was built: a real self-hosted dashboard, no cloud account required, released open-source.

Reverting to stock solved the Rinkhals-specific bugs immediately — the diamond-squares issue vanished, uploads worked normally again. But it traded away the one thing Rinkhals actually gave: any visibility into the printer at all when not standing in front of it. Anycubic's own path to that is a cloud account and their own app, both deliberately avoided here — not out of pure network-security habit, but because a cloud-connected printer means a print could get sent or resumed by someone other than you, or a fault could sit unnoticed on someone else's server instead of tripping a local alert. That's not an abstract worry on a machine whose nozzle runs at 300°C — a monitoring setup for that has to be something you actually trust, not a round trip through someone else's infrastructure. This is a printer on a home network with its own reverse proxy setup already in place; the fix needed to be local.

### How the real protocol actually works

Anycubic's official apps (Slicer Next, the cloud app) talk to the printer over local MQTT — but they get there via a live handshake first, not a fixed set of credentials. That handshake, and most of the command set, got worked out directly first — about 80% of it before looking for anyone else's prior work at all. Two existing open-source repos got found and used from that point to cross-check and fill in the rest: [rvanderp3/kobra-connect](https://github.com/rvanderp3/kobra-connect) (the most complete public MQTT command reference found anywhere) and [SlimQuiggle/KobraCache](https://github.com/SlimQuiggle/KobraCache) (independently confirmed the file list/delete command shape, working live against a different Kobra model):

1. A plain HTTP call to the printer's own local API (port 18910) returns a token.
2. That token, combined with a timestamp and a nonce, produces a signed request for per-session credentials.
3. The response is AES-CBC encrypted; decrypting it (with key material derived from the original token) yields a broker address, a username/password, and a client certificate — all specific to that one printer, generated fresh every session.
4. From there it's a normal mutual-TLS MQTT connection, subscribing to the printer's own status topics and publishing commands (pause, resume, light on/off, temperature, fan, print-speed mode, filament/drying control, file listing) to its command topics.

> **Nothing hardcoded, and that mattered.** Every session performs its own live handshake against the printer's IP — there's no shared secret embedded anywhere in the code, in this project or the two referenced above. That distinction is what made open-sourcing the project reasonable at all: the protocol itself is already public knowledge across a small handful of independent community efforts (this one included), none of which embed Anycubic's shared fleet credentials. The one hard rule that stayed firm the whole way through: real shared/fleet-level secrets never go in a public commit, even though the protocol and the client code are fine to share.

### What actually shipped

Every feature below is confirmed against a real, running printer — not just "the code compiles" confirmation, actual "the commanded value changed the real machine" confirmation:

- Live status: state, progress, layer, ETA, nozzle/bed temps, fan speed, filament slots
- A genuine continuous camera stream (not polling), independent of Slicer Next needing to be open
- Pause / Resume / Emergency Stop
- Full file browser (local storage + USB): list, navigate, delete, thumbnail preview, upload, start a print
- ACE box: filament colours/types per slot, active-slot indicator, drying on/off
- Live controls: light on/off, nozzle/bed temperature, fan speed, print-speed mode mid-print

> **A camera that isn't the official one, works fine.** A generic USB webcam (Microsoft LifeCam HD-3000, 720p) works cleanly on completely stock firmware, no hacking required — despite Anycubic's own documentation implying you need their specific camera module. A higher-resolution camera (Razer Kiyo, 1080p) was also detected and streamed, but with a "doubled frame" artifact — reproduced identically in Anycubic's own Slicer Next too, so it's the printer's onboard video pipeline mishandling the higher resolution, not a fixable client-side bug. A 720p webcam is the safer bet until this gets more data points.

Getting here wasn't friction-free. A silent stale-connection bug (the dashboard would keep showing "Connected" with hours-old data after the socket had actually died, because nothing was probing it) only surfaced during a genuine multi-hour soak test — exactly the scenario the app exists for. Fixed with an explicit MQTT keepalive plus an application-level staleness watchdog, then re-confirmed with a full overnight soak test before calling it done.

---

## Chapter 3: The Calibration Wizards, and the Bug With No Error Message

> **TL;DR** — Slicer Next's built-in temperature, retraction, and max-flow calibration wizards all silently print one flat value instead of a real sweep, with zero warning. The cause: repairing a mesh's "manifold errors" after setting up the test wipes the calibration settings, because repair rebuilds the object under a new internal ID. Don't repair — just set the range and slice.

This is the bug that ate a full night. Every calibration attempt — temperature tower, retraction tower, max-flow test — came out looking identical from top to bottom, no visible stepping, no error message anywhere to explain why.

### The trap

Slicer Next (an Anycubic-branded fork of OrcaSlicer — confirmed literally, some bundled files still carry their original OrcaSlicer names) supports per-height "Height Range Modifier" settings, letting one model print with different temperatures (or other parameters) at different heights. That's exactly how a calibration tower works: a temperature tower is really just a tall test print with a different nozzle temperature commanded every 10mm.

Importing a calibration STL commonly throws a "non-manifold" warning — an apparent invitation to click Repair. Doing that *after* the height-range settings were already configured is what silently breaks everything: repair regenerates the object under a new internal ID, and the height-range configuration stays bound to the old one. The print still runs. It just quietly falls back to a single flat value, because nothing is actually attached to what gets sliced anymore.

> **The actual working recipe.** No custom G-code, no rebuilding files by hand — the real fix is almost embarrassingly simple once you know it:
> 1. Open the calibration model (e.g. `temperature_tower.stl`).
> 2. Set the range/step directly (e.g. 210–240°C, 5° steps).
> 3. Slice.
> 4. In Preview, switch the colour-coding dropdown from **Line Type** to the parameter being tested (e.g. **Temperature**) and confirm multiple distinct colour bands — that's your proof it's actually varying.
> 5. Send and print directly.
>
> **Ignore the non-manifold error count on import.** One working run had 8 reported errors and printed perfectly. The only thing that actually matters is never clicking any repair action once the range is configured. Both `temperature_tower.stl` and `retraction_tower.stl` turned out to already be fully manifold anyway (confirmed via direct mesh analysis, zero non-manifold edges on either) — repair was never actually necessary for either one.

Confirmed against real hardware, not just theory: a temperature transition landed at exactly the predicted layer — 235°C commanded precisely at layer 51, the exact boundary a 10mm block at 0.2mm layer height should produce. The math and the workflow both checked out precisely, not approximately.

### The max-flow test's extra wrinkle

The max-volumetric-flow test uses a continuous vase/spiral toolpath — Z climbs on almost every single line, not in discrete layers. A naive check of the periodic layer-comment markers in the gcode can read as "flat" even when the real per-line feed rate is genuinely ramping, because those markers don't fire on every line the way they do in a normal layered print. The only reliable check is the actual per-line Z/F values in the raw gcode — a genuinely-orphaned flat result and a real, working sweep can look deceptively similar from the printed object alone.

### Building an alternative, in parallel

Frustration with fighting Slicer Next's format led to starting a completely independent calibration tool from scratch — a small Windows app that generates calibration test G-code directly as simple parametric geometry, no real slicing engine involved at all, since calibration prints are just repeating shapes. The temperature-tower module got built and worked; retraction, flow, and max-flow modules were scaffolded but not finished, because the real Slicer Next fix was found in parallel and made the workaround less urgent. It's still sitting there as a reference tool for calibration parameters, not a finished product — sometimes the thing you build out of frustration isn't the thing you end up needing.

### Two real lockups, and what caused them

Trying OrcaSlicer directly (rather than Anycubic's Slicer Next) as an alternative path seemed reasonable — until it hard-locked the printer, twice, both requiring a full power cycle to recover. Neither attempt got a parse error or any visible rejection; the printer simply stopped responding to everything, including basic network reachability.

> **The real cause: a command flood, not a bad file.** An offline command-frequency diff between a known-good Slicer Next export and an OrcaSlicer export of the exact same model found it: OrcaSlicer's Klipper G-code backend emits the native macro `SET_VELOCITY_LIMIT` on every single print-feature change — walls, infill, travel — **8,226 times** in one ~350-layer print. Slicer Next's real output calls the equivalent macro **exactly once**, using the Marlin-style `M204 S<value>` instead for all per-feature acceleration control. The printer's firmware almost certainly can't handle that command at that call volume — a full lockup, not a rejected/invalid command, points at a buffer or queue bug rather than a parser one.
>
> A second identical-model comparison reproduced the exact same pattern, ruling out "unusual geometry" as the explanation. The command itself isn't forbidden or unrecognized either — it's a genuine, intentionally-exposed controller method in the firmware's own API. It's specifically the volume of rapid-fire calls that nobody designed the firmware to handle.

The practical rule that came out of this: never send an OrcaSlicer-generated file — raw G-code or a repackaged `.3mf` — to this printer without translating its dialect first. A working translator was built and verified offline (rewriting `SET_VELOCITY_LIMIT` calls to `M204`, `SET_PRESSURE_ADVANCE` to `M900 K`, inserting the acceleration/jerk-limit commands OrcaSlicer never emits at all) — a translated file was tested live afterward and, critically, did not lock up the printer, unlike either untranslated attempt.

---

## Chapter 4: Reading the Real Firmware

> **TL;DR** — Anycubic's official `.swu` firmware update files are encrypted with a publicly-known password (documented by the Rinkhals project) — decrypting one is completely offline, zero-risk, and answers questions no official documentation covers at all: real safety temperature limits, the real upload pipeline, and the actual mechanism Rinkhals itself uses to install.

Two things made this worth doing directly rather than guessing: the ongoing upload-path mystery (files were getting accepted by the printer but never actually appearing in its own file list), and general curiosity about what the wizard bug and the OrcaSlicer lockup actually shared underneath.

### Getting inside, safely

A `.swu` file is a plain ZIP containing one more ZIP (`update_swu/setup.tar.gz`) that's ZipCrypto-encrypted using a password that's already public knowledge in the Rinkhals community. Decrypting it needs nothing more than Python's own built-in `zipfile` module and that password — no custom tooling, and critically, nothing that touches the actual printer. This is purely reading a file Anycubic already published for public download.

> **What the firmware actually confirmed.** The `.swu` for the version running at the time only contains the "app" tier — the touchscreen UI, camera service, and local API server, about 83 files total, all Go-language ARM binaries. No Klipper/MCU source in this package; that layer (where `SET_VELOCITY_LIMIT`, the mesh-leveling macros, etc. actually live) is separate and updated less often. What was in this package still settled a lot:
> - Real safe temperature range, straight from the config: **185–300°C** for the nozzle — the stock calibration reference tower's full 170–350°C range genuinely exceeds the firmware's own configured safe bound at both ends.
> - The real gcode storage path (`/useremain/app/gk/gcodes/`) and the local API's real port (18086, loopback — the LAN-facing port the dashboard actually uses is a separate gateway in front of it).
> - Port 80 hosts a real, live JSON API — but with no REST-style URL paths anywhere in the binary's strings, pointing at a Go `net/rpc`-style single-endpoint dispatcher (method name inside the request body) rather than per-action routes.

### Solving the upload mystery, from the actual source

Go binaries keep readable function and string names unless deliberately stripped — which these weren't. A short script scanning for runs of printable characters turned three compiled binaries into a surprisingly complete picture of the real upload/print pipeline, replacing everything that had been inference or guesswork up to that point:

- The real upload endpoint takes no destination path parameter at all — the `root=local`/`path=` fields tried earlier were harmless guesses that Go's HTTP server just silently ignored, not the actual fix.
- An uploaded package gets extracted to a staging directory, run through a "3MF precheck" (which can fail on insufficient space), then searched for a matching `plate_*.gcode` file inside — preferring `Metadata/plate_1.gcode` specifically, falling back if that exact name isn't found.
- A separate "convert legacy 3mf" step exists too, with its own distinct failure mode — meaning a hand-rebuilt file, even one that's byte-verified correct, can still get classified as "legacy" and fail at a completely different stage than expected.
- "Printer is busy, cannot upload while printing" is a real, distinct rejection — meaning a stuck busy-state left over from an earlier crashed attempt could easily explain later failures that look format-related but aren't.

> **The one line that decodes the update mechanism entirely.** The whole update-validation chain, read straight from the real update script, is one line: unzip the package with the known password into the right directory. No signature check, no certificate, no checksum beyond whatever the unzip step itself does. Anything with the correct password and file layout gets executed as a firmware update. The same script also runs `killall sshd` and deletes the SSH binaries on every single update — the underlying OS fully supports SSH, Anycubic just strips it out every time, almost certainly specifically to fight back against exactly this kind of community modding.
>
> Separately, the USB-drive update trigger folder name is base64-obscured in the script (trivial to decode): drop an `update.swu` into a folder literally named `help_sos_` at the root of a USB drive, insert it, and the firmware detects and runs it automatically — no menu interaction at all. This is almost certainly the exact mechanism Rinkhals itself uses to install.

None of this turned into a shipped addon — the actual plan (a self-hosted web UI running directly on the printer's own storage, reverse-proxied the same way this printer's other self-hosted services already are, replacing the separate always-on PC that Kobra LAN Monitor currently needs) is fully scoped but deliberately not started. The most promising lead for it: a Unix domain socket the firmware's own API config points at, which is very likely Klippy's own standard socket — the same one real Moonraker connects to on any normal Klipper install. If that's confirmed, a future addon wouldn't need to reverse-engineer the printer's proprietary API at all, just speak the same well-documented interface Moonraker already does.

---

## Chapter 5: My Confirmed Calibration

> **TL;DR** — Every value below is confirmed against a real, physical print on this one printer, with this one filament — not a slicer default, not a guess, but also not a universal number for every Kobra 3 Max. Different filament, a different unit off the line, or a different environment will all shift these. Treat the table as a worked example of the process and a realistic starting point, not a number to copy in blind — run the same sweeps on your own machine and filament before trusting a print to them.

| Parameter | Value | Notes |
|---|---|---|
| Nozzle temperature | **240°C** | Real 210–240°C sweep, 5° steps — confirmed live, block transitions landed exactly on the predicted layer. |
| Flow ratio | **0.915** | From a real flow tower — no visible over/under-extrusion, consistent wall thickness. |
| Pressure advance | **0.05** | PA Line method, direct-drive extruder. Governs corner timing only, not adhesion. |
| Retraction distance | **0.2mm** | Real 0.1–2.0mm sweep — stringing only below 0.2mm, genuinely low because this is direct-drive. |
| Max volumetric flow | **>15.7mm³/s** | Real continuous-ramp test (~29–112mm/s) — perfect quality throughout, ceiling never actually found. |
| Speed modes | **All 3 OK** | Silent / Standard / Sport all confirmed live — stock cornering values hold at every speed. |
| First layer height | **0.20mm** | Kept fixed regardless of body layer height — see the trap below. |

> **Settings can silently revert on unrelated edits.** Changing body layer height from 0.20mm to 0.12mm — the *only* field touched — silently reset retraction back to the old 0.8mm default, with no warning at all. Same underlying class of bug as the mesh-repair issue above, just a different trigger. After any profile edit, including a plain layer-height change, re-open the retraction and filament settings and confirm temperature, pressure advance, flow, and retraction all still match the table above before trusting a print.

> **A diagnostic trap: over-extrusion that isn't.** What first looked like real over-extrusion at 0.12mm layer height turned out to actually be First Layer Height silently reverting to match the new 0.12mm body height instead of staying at the separately-set 0.20mm — a third casualty of the same layer-height-change bug that already hit retraction. A too-thin first layer drags and scrapes rather than laying down cleanly, and that dragging visually mimics real over-extrusion closely enough to send you chasing completely the wrong setting. **Before touching flow rate after any layer-height change, confirm First Layer Height first** — it should generally stay around 0.2mm regardless of how thin the body layers get.

---

## Chapter 6: A Second Mystery: Blobby Text, Wrong Lever

> **TL;DR** — Small embossed text on multi-loop objects (spool labels, part numbers) came out blobby on every single piece — but retraction, seam position, and even bed adhesion were all red herrings. The real cause was a non-default wall-print-order setting, found by reading the actual slicer profile JSON directly rather than guessing through the UI.

Well after the main calibration was confirmed clean, a completely separate print — a set of small filament-spool ID labels, and later a multi-part organiser with embossed diameter labels — came out with a blob concentrated at roughly one spot on every single piece, plus a matching split in the surrounding skirt outline.

Retraction distance was the obvious first suspect (many small islands means many more retract/prime cycles than the calibration tower ever tested) — bumping it from the confirmed 0.2mm to 0.5mm as a "meet in the middle" test changed nothing. Seam position was already set to Random. Neither was the actual lever.

> **The real cause, found by reading the actual profile file.** Rather than keep guessing through the UI, reading the process profile's saved JSON directly (Slicer Next stores per-profile overrides in plain, readable files) surfaced it immediately: `wall_sequence` was set to **"Inner/Outer/Inner"** instead of the standard two-step order. That sequence sandwiches the visible outer wall between two inner passes — meaning every single closed loop in the model, including each tiny letter-stroke in embossed text, gets an extra start/stop transition compared to the normal order. More transitions per shape means more chances for a seam/wipe artifact to show up, concentrated right at each loop's restart point — which is exactly the pattern in the photos.
>
> Alongside it, `filament_max_volumetric_speed` was still sitting at 200mm³/s — a value deliberately raised during the max-flow calibration test itself (so the sweep wasn't artificially capped by the stock 13mm³/s default), but never brought back down to a real number afterward. Left at 200, it effectively disables the slicer's own protective flow-rate limiter entirely. Corrected to 20mm³/s — comfortably above what the current wall speeds actually need, but a real number again instead of "off."

> **One more, purely physical lesson.** A separate thin-ring-shaped part failed for a completely different, much simpler reason: not enough bed contact area for reliable first-layer adhesion. The fix wasn't a slicer setting at all — a proper bed clean (soap, then isopropyl alcohol, then a fresh glue-stick layer) solved it outright. Worth remembering: not every print-quality problem is a calibration problem. Sometimes the plate's just dirty.

> **Also worth knowing before chasing a settings problem.** If a defect is confined to a specific part of the object or a specific gcode feature type rather than spread evenly throughout, general settings that apply to every layer (wall order, seam, retraction) are the wrong first place to look — check what's actually printing at that exact point instead. Some embossed-label text later turned out to be entirely first-layer content (printed directly on the bed, not distributed through the print), which reframes the whole troubleshooting direction: first-layer squish and bed-texture transfer become the live suspects, not anything that governs the rest of the print.

---

## Chapter 7: The Tools This Left Behind

Nothing here started as a plan to "build tools" — each one exists because a specific, real gap kept getting in the way.

### [Kobra LAN Monitor](https://github.com/A-to-PC/kobra-lan-monitor) — Released

Self-hosted web dashboard via the reverse-engineered local LAN/MQTT protocol. Live status, continuous camera stream, full file management, ACE filament/drying control, live controls — all confirmed against real hardware, no cloud account anywhere.

### [Kobra Time Lapse](https://github.com/A-to-PC/Kobra-Time-Lapse) — In testing

Watches the same LAN protocol for print state on completely stock firmware — no Rinkhals, no Moonraker — and grabs frames from any RTSP camera automatically while a print runs, assembling the finished timelapse the moment it's done. Includes an optional, log-only frame-comparison failure check.

### [3D Time Lapse](https://github.com/A-to-PC/3D-Time-Lapse) — Stable, not actively updated

The same idea, for the other side of the fork: printers running Rinkhals with Moonraker. Polls Moonraker's own REST API for print state instead of the LAN protocol, so it works anywhere Moonraker already runs — never touches Klipper's config, so a fragile firmware setup can't be made worse by it. Left as-is since moving to stock firmware (Chapter 1) — it works, just isn't where new development is happening right now. That's a "not right now," not a "never": if Rinkhals goes back on, this is the one that reopens.

### Kobra Calibration Generator — Reference tool

Generates calibration test G-code directly as parametric geometry, bypassing Slicer Next's format entirely. Built out of frustration mid-investigation; now mainly useful as a reference for calibration parameters rather than an end-to-end pipeline. (Not published as its own repo.)

*Kobra Time Lapse and 3D Time Lapse look almost identical on the surface, but they're genuinely separate tools for two different setups, not two versions of one app — pick whichever matches your firmware.*

### A tripwire, not a trained eye

The failure-detection feature in Kobra Time Lapse deserves an honest caveat, because it's easy to oversell: it compares each new frame against the last one using structural similarity — brightness, contrast, and pattern correlation across small patches of the image — and flags anything that changes by more than a set threshold. That's it. There's no trained model, no understanding of what a 3D print or a print failure actually looks like, unlike a purpose-built service like Obico, which has years of labeled real-failure training behind it.

A real print failure (spaghetti, a knocked-over part, a piece detaching) does cause exactly the kind of sudden large visual change this catches. So does an enclosure light changing, a roller door opening, or the ACE arm swinging into frame. The algorithm can't tell those apart — it's a blunt, generic tripwire, deliberately kept log-only (not automatically pausing anything) until it's been run against enough real prints on the specific enclosure/lighting/camera setup to know what its actual false-positive rate looks like. That tradeoff is exactly what let it get built as an evening's addition instead of requiring a trained model.

> **Why not just use Obico?** Obico's real integration path (`moonraker-obico`) hard-blocks on a genuine Klipper/Moonraker connection from its very first step, and drives everything through Klipper's specific internal object model. On stock firmware, with no Moonraker to poll, that's not a light integration to add — it's either running Rinkhals again just for this one feature, or building a full server that impersonates enough of Moonraker's real API to fool Obico's client, which stays fragile against every future Obico update. Building a smaller, honestly-scoped detector directly was the better trade for this setup.

---

## Quick Reference: Every Practical Tip in One Place

If you only read one section, read this one.

- [x] **Never repair a calibration model after setting up its height ranges.** Just set the range and slice. Ignore the non-manifold error count on import — it doesn't block a correct print.
- [x] **After any profile edit — including just a layer-height change — recheck everything.** Temperature, pressure advance, flow ratio, retraction, and First Layer Height can all silently revert with zero warning.
- [x] **A too-thin first layer looks exactly like over-extrusion.** Confirm First Layer Height (should stay ~0.2mm) before chasing flow rate after any layer-height edit.
- [x] **Never send an OrcaSlicer-generated file (raw gcode or .3mf) to this printer untranslated.** Its Klipper output floods a command the firmware can't handle at that volume — two real lockups confirmed this.
- [x] **Blobby text/detail on small multi-loop objects → check wall order before retraction.** "Inner/Outer/Inner" adds an extra transition to every loop. Standard "Inner/Outer" fixed it outright.
- [x] **Set a real max volumetric speed after calibration, don't leave the test's raised ceiling in place.** A value like 200mm³/s effectively disables the slicer's own flow-rate safety net.
- [x] **A generic USB webcam works fine on stock firmware.** No official camera module required. Lower resolution (720p) streams more reliably than 1080p on the printer's own video pipeline.
- [x] **Thin, small-footprint parts need real bed prep, not just calibration.** Soap, then isopropyl alcohol, then fresh glue stick — solved an adhesion failure no slicer setting could.
- [x] **A "success" message from an app or slicer means nothing for real hardware.** Only the printer's own screen, sound, or visible behaviour counts as confirmation — this bit multiple times during upload-path debugging.
- [x] **If ACE Pro auto-backup keeps swapping colours mid-print, check the printer's own backup setting.** Left on its default, it substitutes by material type only, not colour — tightening or disabling it in the printer's own settings, not a firmware bug, is what fixed it here.

---

*Anycubic Kobra 3 Max · stock firmware · living reference — update as new findings turn up.*

## License

MIT — see [LICENSE](LICENSE).
