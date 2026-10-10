# Plan: CarPlay on the Virtual Cockpit of an Audi A6 (C8) with `MH2p_ER_AUG35_P2711`

Target: show CarPlay navigation on the Audi Virtual Cockpit (VC) of an A6 C8 whose
MMI reports `MH2p_ER_AUG35_P2711`. Two separate deliverables fall out of the
existing code, and they have different risk:

| Path | What the driver sees on the VC | Code involved | Status today |
|---|---|---|---|
| A. Turn-by-turn over BAP | Native arrow, distance, road name, lane bar in the VC's own navigation widget | `dio_cluster.so` (iAP2 Route Guidance hook) → PPS file → `CarPlayClusterIntegration` → `CombiBAPServiceNavi` | Works on Porsche; opt-in on Audi, never run on an Audi |
| B. CarPlay map video | The MMI screen mirrored into the VC map area | `cluster` daemon `capture=display` → window bound to displayable `33` | Works on Porsche; untested on Audi, display heuristic likely wrong on an A6 |

Path A is the one to aim for first: it only needs services that already exist
on every MH2p HMI, and a failure leaves the VC blank rather than crashing a
process. Path B is a stretch goal with two known A6-specific problems.

Everything below is ordered so that each step can be undone with
`uninstall.txt` or `Update/rollback.sh`, and so that nothing is compiled until
the logs prove the shipped binaries load on this firmware.

---

## 0. Facts that shape the plan

- **P2711 is the untested firmware train.** The installer accepts `27xx` for
  `OEM=AU` but prints a warning; the hooks were built against Porsche
  `P2870`/`P2874` and the closest Audi match is `P2873`. Upstream issue #8 was an
  A6/A7 on exactly `P2711` and was closed without a result. Expect iteration.
- **The shipped binaries are newer than the sources in this repo** (README,
  "open questions"). The `builds/*_audi.zip` JAR uses the `carConfig` schema
  with nested `mirror` / `gal_h264` blocks, while `lsd/` still parses
  `mirrorCarConfig`. Any change that needs a recompile must either get the
  current sources from upstream or accept a regression. The plan therefore
  puts every no-recompile change first.
- **The CarPlay hook only sees wireless CarPlay.** It is preloaded into
  `dio_manager`, which hosts the wireless iAP2 session; a wired session runs in
  `iap2connectionmanager` and never reaches it. The A6 C8 has wireless CarPlay
  with MMI Navigation plus, so test wirelessly.
- **The CarPlay Java path never hides the native VC map.** The Android Auto
  path calls `IMapClusterService.switchKombiMapToHiddenContext()` before posting
  its window on displayable `33`; `CarPlayClusterIntegration` has no such step
  and only writes `start`/`stop` to the mirror FIFO. For path B this has to be
  added (Java change, see §5).
- **The cluster daemon picks displays by size, not by id.** `src/cluster.c`
  takes the *widest* display as the MMI screen and the first narrower one as
  the cluster. On an A6 C8 the VC is 1920 px wide while the upper MMI display is
  narrower, and there is a second (lower) MMI display, so the heuristic can
  capture the wrong screen. Needs a command-line override (C change, see §5).
- **Shadowed stock classes.** With `enable_carplay.txt` the full JAR is
  installed, which shadows `CarPlayDSIManager` (10-argument constructor) *and*
  `AndroidAuto2Subsystem` (needs the Porsche-only `StorageMountHandler`). For a
  CarPlay-first test the JAR should shadow as little as possible (§3).

---

## 1. Pre-flight on the car (no files changed yet)

1. Confirm the head unit string over the MMI version screen or SSH:
   `MH2p_ER_AUG35_P2711` → `oem=AU type=G35 region=ER sw=2711`. If it is
   `K2470` or lower, update the MMI first; if it is `3xxx` it is MIB3 and out
   of scope.
2. Confirm wireless CarPlay works stock, and that Android Auto is activated if
   path B (which reuses the AA cluster daemon) is wanted.
3. Set the VC to the large-map layout (VIEW button) so the Kombi map
   displayable is actually shown; nothing in the mod can force this.
4. SSH access is strongly recommended (fifthBro/mh2p-ssh-access) because
   `dio_cluster.log`, `gal_preload.log` and the daemon log live in `/tmp`.
5. Copy `Mods/ClusterIntegration/failsafe.sh` to a second SD card's root and keep
   it in the glovebox: it runs `uninstall.sh` at boot if the unit stops
   coming up cleanly.

---

## 2. Diagnostic pass (shipped binaries, CarPlay hook in log-only mode)

Goal: answer four questions before anything is "live".

| Question | Where the answer is |
|---|---|
| Does the shadowed `CarPlayDSIManager` load on P2711? | `cluster.log` on the SD root: `CP: SYS: CarPlay Cluster Integration Initialized`, `SYS: CombiBAPServiceNavi service AVAILABLE`, `DSI: registerDSIListener OK`. If CarPlay no longer starts, the constructor differs. |
| Does the Audi `libesoiap2` export the five symbols the hook interposes? | `/tmp/dio_cluster.log`: `MessageView ctor = 0x...` non-zero and `deployMessage: captured CIAP2ControlSession`. |
| Does the iPhone accept the patched Identify? | `Ident::process: msgid=0x1d02 (ACCEPTED)` after `IDENTIFY PATCHED`. `REJECTED` on every cycle means the Audi already advertises route guidance or the TLV layout differs. |
| Do RG messages arrive and reach Java? | `sendToSession[...]: msgid=0x5201 (RG_Update)` in the hook log and `PPS:` / `RG_BAP:` lines in `cluster.log`. |

Steps:

1. Build the package: `modkit/build_sd_package.sh builds/ClusterIntegration_v0034_beta2_candidate_90d0b76.zip builds/<out>.zip`
   (after the installer changes in §3 are made).
2. On the card create `Mods/ClusterIntegration/diag.txt` **and**
   `Mods/ClusterIntegration/enable_carplay.txt`. In diag mode the installer sets
   `enableMapRender=false`, the `gal` hook does not inject, and the
   `dio_manager` wrapper exports `DIO_CLUSTER_LOG=1`.
3. Install, reboot, connect the iPhone wirelessly, start Apple Maps
   navigation, drive a few hundred metres. The VC should already show arrows
   and distances through BAP in this pass if everything lines up.
4. Pull `Backup/hmi_reference/*.jar` from the card and run
   `modkit/check_hmi_signatures.sh Update/ClusterIntegration_v0034_*.jar <that dir>`
   on a PC. `MATCH` for `CarPlayDSIManager` is the green light for going live;
   `MISMATCH` means the JAR needs a rebuild against the P2711 stock jar (§5).
5. Report back the `CAR_VARIANT:` log line (gives the `carClass_generation`
   key for the A6) and the `disp[n]: id=.. size=..` lines from
   `cluster capture=display verbose=1` over SSH.

---

## 3. Installer and packaging changes (no recompilation, do these first)

All in `modkit/`, picked up by `build_sd_package.sh`.

1. **CarPlay-only JAR variant.** Extend the Python block in
   `build_sd_package.sh` to also derive `Update/cp_only/<name>_cp.jar` that drops
   `de/audi/app/terminalmode/smartphone/androidauto2/**`. The two packages have
   no cross-references (already verified for the `aa_only` derivation), so this
   needs no compiler. Add a marker `carplay_only.txt` to `install.sh`: when
   present on Audi, install the `cp_only` JAR, skip the `gal` wrapper and
   `gal_cluster.so`, and skip the `StorageMountHandler` gate (that class is only
   constructed by the AA subsystem). This shrinks the blast radius of the first
   CarPlay test to one shadowed class and one preloaded process.
2. **Symbol check for the CarPlay hook.** Mirror `check_hook_symbols()` with a
   `check_dio_hook_symbols()` that finds `libesoiap2*.so` under `/mnt/app` and
   greps the five mangled names (`deployMessage`, `sendToSession`,
   `ModuleIdent::process`, `ModuleIdent::identificationInformation`,
   `CIAP2ControlSessionMessageView` ctor). A missing one means the hook is a
   silent no-op on this firmware; print it in the ModKit log.
3. **Stage the hook log to the card.** `install.sh` prints "Diagnostics:" but
   `dio_cluster.log` is only in `/tmp`. Have the `dio_manager` wrapper, when the
   `diag_mode` marker exists, also tee `/tmp/dio_cluster.log` to
   `/fs/sda0/Logs/` on exit (append-copy, best effort) so a user without SSH
   can read it.
4. **Firmware message.** Change the `27xx` warning text to name the A6/A7 case
   explicitly and point at this document, and record `P2711` in the README's
   firmware list as "under test".
5. **`config_overrides.example.txt`.** Add commented entries that matter for an
   A6: `forceRHD=true` (only for UK/IE cars), `enableMapRender=false` (keep
   turn-by-turn only while path B is unresolved), `bargraphMode="distance"` in
   case the Audi widget draws the bar badly.

Validation on a PC: `bash -n` on every ksh script, run `build_sd_package.sh`,
unzip the result and confirm `aa_only/`, `cp_only/` and the markers' behaviour
by reading the installer with `MOD_PATH` pointed at the extracted folder in a
dry-run (the scripts only touch `/mnt/app`, so a dry-run means reviewing the
gate logic, not executing it).

---

## 4. Go-live for path A (turn-by-turn)

1. Remove `diag.txt`, keep `enable_carplay.txt` (and `carplay_only.txt` if
   path B is not wanted yet), re-run the installer. It is idempotent and puts
   the shipped config back.
2. If the diagnostic pass showed `REJECTED` Identifies: do not go live. Delete
   `enable_carplay.txt`, re-run (the installer restores the stock
   `dio_manager`), and move to §5.2.
3. Acceptance: Apple Maps / Google Maps / Waze on CarPlay → VC shows maneuver
   arrow, distance countdown, next road, lane guidance; native MMI navigation
   still works after the phone disconnects (`RG_BAP: doStopRouteGuidance`).
4. Tune through `config_overrides.txt` only: `heartbeatInterval`,
   `bargraphMode`, `imperialSmallUnit`, `forceRHD`.

---

## 5. Changes that need a rebuild (only if §2 or §4 fails, or for path B)

Prerequisites: the current upstream sources for the shipped v0034 build (ask
fifthBro, or diff `lsd/` against a decompile of the shipped JAR), the stock
P2711 HMI jars from `Backup/hmi_reference/`, a JDK with `-source 1.4 -target 1.4`
(or the class-file version the stock jars use), and the QNX 6.6 ARMv7
toolchain (`qcc -Vgcc_ntoarmv7le`) for C.

1. **`CarPlayDSIManager` constructor mismatch** (if `check_hmi_signatures.sh`
   says `MISMATCH`). Re-derive the class from the P2711 stock jar (decompile
   with CFR as the original was), re-apply the three mod touch points
   (`clusterIntegration` field + `init()` block, `onCarplayTerminated()` in the
   terminate path, the `updateMode` resource forwarding), rebuild. This is the
   single most likely blocker on P2711 and cannot be worked around in scripts.
2. **Identify rejected by the phone.** Compare the `IDENTIFY orig` TLV dump with
   the Porsche one. If the Audi Identify already contains `0x5200..0x5204` or a
   `0x001E` component, make `rebuild_identify_with_rg()` skip msgids that are
   already present and skip the component when one exists. If the layout
   differs (no `40 40` marker at the view's begin pointer), the
   `CIAP2ControlSessionMessageView` offsets in `get_msgid()` need re-deriving
   from the Audi `libesoiap2.so` disassembly.
3. **Wired CarPlay support** (optional). Add a `/proc/self/comm` gate and
   allow preloading into `iap2connectionmanager` too; the inject trigger there
   is the `0x5703` WiFi-config response rather than `0xFFFB`, as the source
   comments already note. Only do this after wireless works.
4. **Path B, Java side.** In `CarPlayClusterIntegration.handleScreenOwnerChange()`
   call the same `IMapClusterService` hide/show pair the AA path uses (via
   reflection, as `AndroidAutoClusterIntegration.callMapClusterService()` does)
   before `sendMirrorCommand("start")` and after `"stop"`, and track the
   service through a `ServiceTracker` the way `AndroidAutoClusterActivator`
   does. Without this the native Kombi map and the mirror window fight for
   displayable `33`.
5. **Path B, daemon side.** Add `hmi_disp=<id>` and `cluster_disp=<id>` arguments
   to `cluster.c` so the capture source and the output display are chosen by
   `SCREEN_PROPERTY_ID` instead of by width, and make `DISP_ID_TARGET` /
   the `"33"` id-string configurable in case the A6 HMI uses another displayable
   for the Kombi map. Feed the values from a new `carConfig` entry for the A6
   (`"<carClass>_<generation>"` from the `CAR_VARIANT` log line) with a
   `mirror` block tuned for a 1920x720 VC.
6. **Mirror framing.** The mirror captures the whole upper MMI display, so
   CarPlay's status bar and the MMI bezel land on the VC. Start with
   `mirror.mode="fill"`, then dial in `zoomX/zoomY/panX/panY` with
   `cluster capture=test ...` over SSH.

---

## 6. Rollback matrix

| Symptom | Action |
|---|---|
| CarPlay no longer connects after install | `enable_carplay.txt` removed + re-run installer (stock `dio_manager` back) |
| CarPlay UI fine, no arrows on VC, hook log shows `REJECTED` | same as above, then §5.2 |
| CarPlay UI fine, `cluster.log` has no `CP:` lines | shadowed class did not load → `uninstall.txt`, then §5.1 |
| Unit boot-loops or MMI blank | `failsafe.sh` card, or `uninstall.txt` |
| VC shows garbage / wrong screen after path B | `enableMapRender=false` via `config_overrides.txt`; keep path A |

---

## 7. Order of work and effort

| # | Item | Needs | Effort |
|---|---|---|---|
| 1 | §3.1 `cp_only` JAR + `carplay_only.txt` | scripts only | small |
| 2 | §3.2 `check_dio_hook_symbols` | scripts only | small |
| 3 | §3.3 log staging, §3.4 messages, §3.5 example overrides, README | scripts/docs | small |
| 4 | §2 diagnostic pass on the car | car + SD card | one drive |
| 5 | §4 go-live, path A | car | one drive |
| 6 | §5.1 / §5.2 only if step 4 fails | JDK or QNX toolchain + upstream sources | medium to large |
| 7 | §5.4 + §5.5 + §5.6 path B | both toolchains + upstream sources | large |

Steps 1 to 3 can be done in this repository now and shipped as a new
`builds/*_audi.zip`. Steps 4 and 5 need the car. Steps 6 and 7 are blocked on
obtaining the sources that match the shipped binaries.
