## Ugoos AM9 Pro / S905X5-J: Dolby Vision P8.1 motion smearing

DV playback has visible smearing/trailing in motion scenes, mainly on faces and high-contrast edges. SDR is clean on the same setup.

Device - Ugoos AM9 Pro, S905X5-J/S6, 4 GB  
CoreELEC - custom `22.0-Piers_devel_20260712122648`, kernel `5.15.196`  
Display - LG OLED C5, direct HDMI, no AVR/eARC  
Content - streaming DV P8.1, 3840x2160/24; confirmed in log  

Captured run - Player-led / LLDV  
S6 HDMI path - `2160p60`, `YUV422`, `12-bit`  
Decoder - 4K 10-bit HEVC, `dynamic_buf_num_margin=12`  
DIM/deinterlacing - bypassed  

Frame trace - 8 `drop:1` frames during initial DV modeset, 14 in a later transition, 1301 `drop:0` frames between them. No Dolby/VDEC failure during steady playback.

Tested - TV-led and Player-led; 24 Hz and 60 Hz; same symptom. Refresh-rate switching and delay; no change. Kodi noise reduction disabled; no change. TruMotion on and off; no change. A certified HDMI 2.1 cable that works correctly with a PC on the same TV input; no change. Circulated S6 `dovi.ko` variants; no change.

Control - Kodi 22 Beta on the LG C5, installed through LG Developer Manager, has no artefact in the same title/scene or other tested streams. Kodi under stock Ugoos Android shows a similar symptom.

This is not a FEL seek-crash report. Is there a known S6 Dolby/VPP issue, or a supported diagnostic to distinguish the Dolby path from the general HDR output path?

System log - https://paste.coreelec.org/SkinnedBrain

### Instrumented S6 results (2026-07-11)

Custom CoreELEC builds instrumented the S6 Dolby path without enabling Dolby
certification mode.

- 720 display-vsync samples during visible smear: steady interval average
  `16.666671 ms`, standard deviation `0.358 ms`, no intervals above `18 ms` or
  below `15 ms`.
- Dolby control-path runtime: average about `225 us`, maximum below `300 us`;
  no `>=1 ms` events.
- Decoder receive/display counters remained equal and drop count remained zero.
- 288 consecutive source frames: generated and applied settings matched for
  every frame. Average generate-to-apply delay was `0.411 ms`, maximum
  `0.624 ms`; no vframe gaps or adjacent-frame metadata mismatch.
- The decoder exposes pixels and DV auxiliary data through the same `PIC_s`
  object/index. There is no independent P8.1 RPU queue capable of drifting by
  one frame in this path.
- Hardware readback during 718 steady samples matched all 27 software-generated
  Core1 DM registers (`718/718`). Core1/Core3 CRCs changed with video frames.
- Runtime `el_flag=0`, confirming single-layer playback and disabled composer.

One real open-driver regression was found in
[`36e330a264c7`](https://github.com/CoreELEC/common_drivers/commit/36e330a264c778b99fbe71675471716189e3bef1):
`el_halfsize_flag` was used without requiring `el_flag`. Consequently P8.1 was
programmed with `CORE1A_SWAP_CTRL0=0x35` (EL 1/4 mode) despite `el_flag=0`.
Gating half-size mode on `el_enable` corrected runtime programming to
`SWAP_CTRL0=0x01` and bypass control from `0x78` to `0x70`, while hardware DM
readback still matched `598/598`. The visible smear was unchanged, so this is a
valid correctness fix but not the smear root cause.

The remaining boundary is after correct frame/metadata pairing and DM-register
commit, but it is not yet limited to the closed module or hardware. TV-led and
Player-led show the same symptom, consistent with their shared Core1 stage;
HDR10 fallback of the same DV base layer is visually clean.

### Open-source S6/STB 2.6 audit

The original S6 DV bring-up mostly added `is_aml_s6()` to the existing
SC2/S4D/S7D paths. Later S6 fixes established that some hardware semantics do
differ, notably Core3 clock gating and reset behaviour.

A Core1 LUT-change asymmetry was tested in `amdv_hw.c`:

- the STB 2.6 Core2 path explicitly says `CP_FLAG_CHANGE_TC2` may be absent and
  compares the generated LUT with the last applied LUT before deciding whether
  to upload it;
- the Core1A/Core1B path trusts only `CP_FLAG_CHANGE_TC` and has no equivalent
  content comparison;
- both Android and CoreELEC use an STB 2.6 control path on S6.

The content-based fallback was run during visible P8.1 smear. Its diagnostic
never fired and the image was unchanged. The control path was already setting
the change flag or generating an unchanged LUT. Stale Core1 LUT programming is
therefore rejected for this reproduction.

A separate, exact routing regression is present in CoreELEC commit
[`26409b80a3e4`](https://github.com/CoreELEC/common_drivers/commit/26409b80a3e4d9f63edf894ed525f7e4064b83de).
The FEL fix widened the first `dv_core1a_set()` render branch from G12 to
SC2/S4D/TM2/S7D/S6. That makes the following vendor SoC-specific branches
unreachable. On S6, five `AMDV_PATH_CTRL` input/output route fields are no
longer programmed; only the Core1 enable bit is written. Current Amlogic/
Hardkernel source retains the correct `G12`-only first branch and then executes
the full S6 routing branch.

Patch `0005-amdv-restore-s6-core1-routing-programming.patch` restores that
vendor branch structure. It does not guess register values: it re-enables the
existing S6 sequence already present in the driver. The patched source and
runtime marker were verified, then visually tested on the same scene. Smear and
sluggish motion were unchanged. Keep the correctness fix, but do not repeat it
as a smear candidate.

The spatial character of the artefact also changes prioritisation. Tone curves,
CSC and CVM are point transforms and cannot by themselves spread pixels along
motion. With the routing repair visually neutral, remaining work follows the
target-DV pixel and signaling boundary. ABI and partitioned-RDMA remain
secondary until they can explain a spatial error rather than merely stale
colour/tone state.

Requested comparison: run the same short P8.1 sample on another S6 reference
device or Dolby IDK setup and compare generated Core1 DM/LUT hashes and stage
CRCs. A reference S6 `dovi.ko`/expected CRC is needed to distinguish control
path output from Dolby-core hardware behavior.

### Decisive same-stream DV-to-HDR10 A/B (2026-07-13)

The same streamed P8.1 episode and scene was kept on the same AM9 Pro decoder,
HDMI port, cable, LG picture mode and 2160p60 display mode. Only the Dolby
output target was changed at runtime.

- Normal Player-led output: input `DV FORMAT_DOVI`, output
  `DolbyVision-Lowlatency`, `YUV422 12-bit`; visible motion smear and sluggish
  motion.
- Forced HDR10 output: input remained `DV FORMAT_DOVI`, output
  `HDR10-GAMMA_ST2084`, `YUV420 10-bit`; motion was clean and responsive.
- Returning to Dolby Vision immediately restored the same smear and sluggish
  motion. Returning to HDR10 immediately removed both.

The steady 24-to-60 cadence and sampled stage-CRC change counts were
effectively identical in the two modes: Core1 BL `73/46` vs `72/47`, Core1
output `72/47` in both, Core3 input `4/115` in both, Core3 output `38/81` vs
`37/82`. These counts match regular frame repetition and do not identify a
frame-pacing fault.

This rejects storage, decode, RPU extraction, frame/metadata pairing, cadence,
general HDR/VPP processing and LG MEMC as the primary cause. The fault is
selected by `dst_format=FORMAT_DOVI`: target-specific Dolby pixel processing
or HDMI Dolby signaling. It does not prove that the closed control path emits
correct target-DV coefficients, because target HDR10 uses different DM/LUT
tables.

Transport alone is unlikely to explain both Dolby modes: Player-led uses
`YUV422 12-bit DV-LL`, while TV-led uses an `RGB/444 8-bit` Dolby tunnel, and
both reproduce the symptom. The clean HDR10 run used `YUV420 10-bit`, so an
exact matched-transport HDR A/B remains optional, not required to establish
the Dolby-only boundary.

#### Rejected causes / tests not to repeat

- Kodi skipped/dropped frames or decoder queue underrun.
- HDMI-vsync approximation / experimental real-vsync clock.
- RPU IRQ/control-path latency or late metadata application.
- P8.1/FEL composer and enhancement-layer buffering (`el_flag=0`).
- S6 half-size EL state, S6 Core1 route restoration, and Core1 LUT fallback.
- VSR/SAFA, SR, VPP PQ and noise-reduction controls.
- 24 Hz alone, LG TruMotion alone, HDMI cable or TV input.
- Player-led versus TV-led and circulated S6 `dovi.ko` variants.

#### Completed VSIF discriminator

Keep the normal Player-led Core3 pixel payload and `YUV422 12-bit` formatter,
but suppress only the Dolby HDMI VSIF. A direct TPI packet-bank write was tried
first: bank 8 changed from `0xe0` to `0x20`, but the HDMI driver restored it to
`0xe0` within 13 seconds. Therefore that live check was not sustained and its
visual result is invalid. Do not repeat it or use an unsafe register-write loop.

The valid test used build patch
`0006-amdv-add-s6-player-led-vsif-signal-test.patch`. Its default-off parameter
`s6_dv_ll_vsif_signal_off` keeps the Player-led Core3 path, LL callback and
`YUV422 12-bit` AVI configuration, while clearing only `dobly_vision_signal` in
both regular and L11 VSIF layouts. Enable it while idle, then reopen the same
P8.1 scene. Build `22.0-Piers_devel_20260713210518` was installed and the
parameter was enabled while idle.

Observed with the exact same Player-led stream and scene:

- input remained `vd1(inst1): DV FORMAT_DOVI`;
- internal Dolby output remained `IPT_TUNNEL`;
- transport remained `2160p60`, `YUV422`, 12-bit, limited, BT.2020;
- only the HDMI Dolby signal changed: HDMI status became SDR and the TV did not
  show its Dolby Vision mode badge;
- raw vendor InfoFrame bank 8 was
  `81 01 1b 4c 46 d0 00 01 00 ...`, i.e. Dolby OUI, low-latency bit 1 and
  `dobly_vision_signal=0`;
- visible smear and sluggish motion disappeared completely.

This proves that the visible defect requires the sink to enter its Dolby Vision
mode while decoder/cadence and the electrical transport stay unchanged. It does
not by itself prove that the Player-led Dolby pixel encoding is valid: with the
signal bit clear, the TV interprets those same pixels using a non-DV transfer
path. The remaining boundary is therefore the closed S6 Dolby output encoding
versus LG's external-input Dolby pipeline.
The immediate same-build signal-bit-on control confirmed causality:

- with the parameter returned to `N` while idle, the same scene again showed the
  LG Dolby Vision badge and immediately regained the same smear and sluggish
  motion;
- all input, internal Dolby mode and transport values stayed identical;
- active DV-LL raw bank 8 became
  `81 01 1b 4a 46 d0 00 03 00 ...`, versus the clean signal-off packet
  `81 01 1b 4c 46 d0 00 01 00 ...`;
- the only payload change is `dobly_vision_signal: 0 -> 1`; the checksum changes
  accordingly from `0x4c` to `0x4a`.

The A/B is therefore causal, not a build, stream, frame-rate or refresh artifact.
The full packet audit found the active packet internally valid: HDMI VSIF type
`0x81`, version 1, length 27, Dolby OUI, valid checksum, LL=1, signal=1,
source-DM-version=0, and no L11/aux/backlight/game metadata. Its serializer is
identical to the older public hdmitx20 path; bank 8 is correct for the LG EDID's
no-IFDB case. Raw AVI was `82 02 0d f0 72 a8 04 61 00 ...`, matching the parsed
YUV422/BT.2020/limited/VIC97 state. No public S6 VSIF ordering, field or checksum
bug is visible. The practical code path is now a colour-correct DV-input to
HDR10 output, or an experimental LLDV-to-HDR10 signaling mode whose colour
semantics must be validated rather than assumed.

#### LLDV-to-HDR10 signaling experiment

With the signal-off parameter active during Player-led playback, writing `hdr`
to the existing HDMI `config` sysfs node immediately changed status to
`HDR10-GAMMA_ST2084`. The Dolby per-frame update loop then cleared the DRM
InfoFrame again in under two seconds, returning HDMI status to SDR. Repeated
live writes were deliberately not used. A small default-off patch is required
to finish every DV HDMI update with a stable HDR10 DRM InfoFrame while retaining
the Player-led Dolby pixels and YUV422 12-bit formatter.

One raw AVI sample (`72 a8 04`) was captured during a transition and initially
looked inconsistent. Three consecutive steady-state reads were instead
identical at `82 02 0d 90 32 e8 64 61 ...`, matching YUV422, extended BT.2020,
limited range and VIC 97. The stale transition sample is not a sustained-motion
root cause and must not be used as evidence of an AVI-format bug.

Build `22.0-Piers_devel_20260713220223` then added a stable default-off
`s6_dv_ll_hdr10_signal` mode. With AFR enabled, the test state was fully
verified at 2160p24:

- input `DV FORMAT_DOVI`, internal output `IPT_TUNNEL`;
- YUV422 12-bit limited BT.2020, VIC 93;
- Dolby VSIF signal clear (`81 01 1b 4c 46 d0 00 01 ...`);
- valid HDR10 ST2084 DRM packet in bank 6 and an HDR badge on the LG;
- stable HDR10 status rather than the earlier per-frame return to SDR.

Smear and sluggish motion remained. Colour and brightness looked approximately
normal but were not measured. Therefore a signaling-only LLDV-to-HDR10 mode is
not a working fix. Combined with the clean real Dolby-input to HDR10-target A/B,
the defect is now localized to the S6 target-DOVI/LLDV output payload or its
closed control-path/hardware programming, not frame cadence, 3:2 pulldown,
Dolby VSIF syntax or LG entering its Dolby picture mode. The SDR signal-off case
only hid the defect under an invalid transfer-function interpretation.

#### Initial live target-switch observation at matched 24 Hz (not reproduced)

With Kodi AFR enabled, the same active DV stream and motion scene was first
verified in normal Player-led output at 2160p24. Input was `DV FORMAT_DOVI`,
policy/mode/target were `1/1/1`, HDMI was DolbyVision-Lowlatency, and smear plus
sluggish motion were immediately visible.

Without stop, pause, seek, stream change or refresh change, the driver was
switched through its supported class API:

```sh
echo 2 > /sys/module/aml_media/parameters/dolby_vision_policy
echo 3 > /sys/class/amdolby_vision/dv_mode
```

The verified result remained 2160p24 with the same `DV FORMAT_DOVI` input, but
policy/raw-mode/target became `2/2/2`, class mode became HDR10, and HDMI became
HDR10 ST2084. Smear and sluggish motion initially appeared to disappear and
colour looked normal by eye. That visual observation was later tested with a
persistent implementation and did not reproduce at the known control moment.

Build `22.0-Piers_devel_20260713223601` added the default-off
`s6_dv_force_hdr10` experiment. With the knob enabled, normal policy `1`
selected raw mode/target `2/2`, input remained `DV FORMAT_DOVI`, HDMI was real
HDR10 ST2084 at 2160p24, but smear and lag remained. The exact live sequence was
then repeated, producing policy/mode/target `2/2/2`; smear remained. Finally the
new knob was disabled and `2/2/2` was reprogrammed again. Cadence/lag looked
better with AFR active, but smear was still definite after seeking to the known
control moment.

Therefore the earlier immediate "clean" observation at 24 Hz is not decisive
evidence. It does not invalidate the separately controlled 60 Hz target-HDR10
result. The controlled states show:

1. target-DOVI/LLDV pixels + DV signaling: smeared;
2. target-DOVI/LLDV pixels + valid HDR10 signaling: smeared;
3. Dolby input + real HDR10 target at 24 Hz: smeared;
4. Dolby input + real HDR10 target at 60 Hz: visually clean;
5. HDR10 fallback/native HDR10 at 60 Hz: visually clean.

The exact 60 Hz result was reproduced after reboot on build
`22.0-Piers_devel_20260713223601`: AFR off, `s6_dv_force_hdr10=N`, input
`DV FORMAT_DOVI`, 2160p60 YUV422 12-bit HDR10, policy/mode/target `2/2/2`.
At the known control moment both smear and lag were absent. The same real HDR10
target at 2160p24 had definite smear, while cadence/lag was no longer apparent.

The evidence therefore requires two dimensions rather than one root cause:
there is a target-DOVI defect visible even at 60 Hz, and a separate or
interacting 24 Hz HDR/DV output-path defect that defeats the HDR10-target
workaround. VSIF syntax and the LG merely selecting its DV picture mode remain
excluded as primary causes. The practical clean workaround is currently AFR
off/2160p60 plus DV-input to real HDR10 target; it sacrifices Dolby output and
is not a root fix.

The automatic startup path was then validated with AFR off. From idle SDR,
policy stayed `1` and `s6_dv_force_hdr10=Y`. Opening the same DV stream selected
2160p60 YUV420 10-bit limited BT.2020 HDR10, while input remained
`DV FORMAT_DOVI` and raw mode/target were `2/2`. The LG showed HDR and the known
control moment appeared free of both smear and lag. Player stop/reopen and
boot-persistence checks were then scheduled before calling this workaround
operationally stable.

The player stop/reopen check then passed: HDR was selected again and the same
control moment remained visually clean. Because the module parameter is
default-off after reboot, `/storage/.config/autostart.sh` was installed to set
only `s6_dv_force_hdr10=1` when the writable sysfs node exists; it does not
change policy, refresh rate or other Dolby settings. Its syntax, executable
mode and immediate `Y` result were verified. A reboot check remains.

Final reboot check passed. The same build returned at 2160p60 in idle SDR,
policy `1`, and the autostart restored `s6_dv_force_hdr10=Y`. Reopening the same
DV stream selected HDR and the known control moment remained free of visible
smear and lag. The workaround is operationally stable across player restart and
device reboot for this reproduction.

A post-VPP VDIN capture is an independent confirmation path, but the public S6
VDIN V4L2 driver requires a custom multi-plane capture helper and is not needed
before the VSIF discriminator.

### Upstream rebase and S6 Core3 transition test (2026-07-17)

Branch `codex/s6-dv-output-path` was rebased from `22.0-Piers_beta1` onto
official `CoreELEC/CoreELEC:coreelec-22` commit `0b9db399be`. This moves Kodi
from `9df1aaf8` to `46c546be` and `common_drivers` from `7df48f4a` to
`adca9e0c`. The complete local patch series `0001` through `0009` was
regenerated and apply-checked in order against the new driver tree.

The intervening upstream Dolby commits are stop/seek race, bounds and crash
fixes. They do not change steady target-DOVI pixel programming. Upstream commit
`df34b7a4` does remove an accidental HDMI behaviour: `fr_hint` no longer
hard-codes QMS-VRR. QMS now defaults to `NONE` and is enabled only by an
explicit device-tree setting. This can restore LG TruMotion when AFR previously
left a 2160p60 QMS transport active, but cannot explain true-DV smear when QMS
is inactive. The final `adca9e0c` YCbCr422 preference affects the default
SDR/HDR path; the TV-led RGB8 and Player-led YUV422 Dolby branches bypass it.

The remaining open-source experiment is tied to a documented Amlogic S6
hardware hazard rather than a generic kernel knob. Vendor commit
`41182baaf843` says that writing zero to Core3 `SWAP_CTRL0` bit 31 can reset
sink-led Dolby vsync and corrupt its CRC. The vendor workaround disabled S6
Core3 clock gating, but the driver still writes the full `SWAP_CTRL0=1` value
on every applied frame.

Patch `0009-amdv-add-s6-core3-transition-only-test.patch` is default-off. On S6
only, it keeps all per-frame Core3 DM registers, metadata, metadata-done latch,
DIAG, CRC and `CLKGATE_CTRL=0x20` writes unchanged. It suppresses only the
redundant final `SWAP_CTRL0=1` write on stable frames. That write is retained on
first enable, reset/power-on, Dolby mode/VSVDB/PPS changes, enable-state changes
and hardware readback mismatch. It deliberately does not guess or force bit 31.
When disabled it executes the original write path without the experimental MMIO
readbacks. After a forced write, mismatch checking is deferred for one update so
that a pre-vsync read cannot be mistaken for a hardware failure.

The existing `/storage/.config/autostart.sh` may still enable the known
DV-input-to-HDR10 workaround after boot. Stop playback and clear every output
experiment before judging the rebased upstream baseline:

```sh
p=/sys/module/aml_media/parameters
for n in s6_dv_force_hdr10 s6_dv_ll_hdr10_signal \
	s6_dv_ll_vsif_signal_off s6_dv_core3_transition_only; do
	[ ! -w "$p/$n" ] || echo 0 > "$p/$n"
done
cat /sys/class/amhdmitx/amhdmitx0/vrr_mode
cat /proc/amhdmitx/hdmi_vrr 2>/dev/null
```

Reopen true DV once with the Core3 test still off. This is the decisive check
for the upstream hard-coded-QMS removal. Only if the verified fixed-refresh DV
baseline still smears should the Core3 A/B below be run.

Record the read-only counters while idle, enable the test, then reopen the same
true-DV 2160p60 scene:

```sh
p=/sys/module/aml_media/parameters
for n in swap_writes swap_skips swap_transitions swap_mismatches; do
	printf 'before_%s=' "$n"
	cat "$p/s6_dv_core3_$n"
done
echo 1 > "$p/s6_dv_core3_transition_only"
```

After 20 seconds, capture:

```sh
for n in transition_only swap_writes swap_skips swap_transitions \
	swap_mismatches swap_readback clkgate_readback; do
	printf '%s=' "$n"
	cat "$p/s6_dv_core3_$n"
done
cat /sys/class/amhdmitx/amhdmitx0/vrr_mode
cat /proc/amhdmitx/hdmi_vrr 2>/dev/null
```

The experiment is active only if `swap_skips` grows while `swap_writes` remains
limited to transitions. A nonzero mismatch count means hardware did not retain
the expected enable state and the code automatically restored the write. The
visual success condition remains real Dolby Vision signaling with correct
colour and metadata, but no smear or sluggish motion; HDR fallback does not
count as a fix.

Partitioned RDMA from Hardkernel's newer S6 tree was also audited. It is active
there and isolates Core3 registers `0x3600-0x36ff`, but it is not a small Dolby
patch: it depends on a new RDMA manager, partition registration and several
cross-subsystem files. The current CoreELEC table preserves insertion order and
does not coalesce `0x3603` (metadata done) with `0x36f1` (Core3 enable). The only
credible discriminator is global-table overflow or an early flush. Before any
large port, capture this short diagnostic during two or three seconds of DV:

```sh
cat /sys/module/aml_media/parameters/g_vsync_rdma_item_count
cat /sys/module/aml_media/parameters/g_vsync_rdma_item_count_max
dmesg | grep -E 'rdma_write_reg.*buf overflow|buf overflow'
echo "0x3603 0x36f0 0x36f1" > /sys/class/rdma_mgr/trace_reg
echo 1 > /sys/class/rdma_mgr/trace_enable
# Play DV for 2-3 seconds.
echo 0 > /sys/class/rdma_mgr/trace_enable
dmesg | grep -E '3603|36f0|36f1'
```

If one handle retains `0x3603` before `0x36f1` with no overflow, a partitioned
RDMA port is a no-go for this smear. Overflow, premature execution or reversed
ordering changes that decision to go.

### External LG/QMS evidence (2026-07-17)

No public report was found that exactly matches AM9 Pro/S905X5-J, LG C5 and the
same DV/HDR moving-pixel smear. The closest independent LG report is a C2 HDMI
Dolby Vision case where Bluetooth audio made frame metadata visibly lag behind
moving objects, while the internal webOS player and HDMI HDR10 were clean. DV
was also clean when Bluetooth was removed, so this proves that an external-HDMI
DV timing defect can exist in an LG path, but its specific trigger does not
match this reproduction. See the [LG webOS forum report](https://forum.webostv.developer.lge.com/t/dolby-vision-out-of-sync-through-hdmi-with-bluetooth-audio-connected-to-tv/14291).

The LG C5 panel itself is not known for transition smearing: measured response
is near-instant, while the expected 24p OLED artefact is discrete stutter rather
than a moving pixel trail. Real Cinema handles 24p judder from internal and
external sources. See [RTINGS C5 review](https://www.rtings.com/tv/reviews/lg/c5-oled)
and [C5 settings](https://www.rtings.com/tv/reviews/lg/c5-oled/settings).

QMS remains a credible explanation for the separate “TruMotion is not working”
part. LG G5 owners report that TruMotion becomes ineffective for 24p while QMS
is active, but works for 50/60p. This is forum evidence rather than an LG design
statement: [AVForums QMS/TruMotion thread](https://www.avforums.com/threads/lg-tv-qms-quickmediaswitching-and-trumotion-with-24p.2552218/).
HDMI documents QMS as a VRR-based refresh transition mechanism; it explains
cadence/judder interactions, not target-DV pixel smear by itself: [HDMI QMS](https://www.hdmi.org/spec2sub/quickmediaswitching).
LG separately documents that ALLM/Game Mode disables TruMotion, but does not say
that QMS is equivalent to ALLM: [LG support](https://www.lg.com/us/support/help-library/lg-tv-troubleshooting-poor-picture-quailty-on-ps5--20153303206885OLT).

There are current AM9 reports for DV/QMS cadence selection, DV black screens,
long picture stalls and FEL seek failures, but not this precise smear:
[24.000 versus 23.976/QMS](https://discourse.coreelec.org/t/fel-amlogic-no-support/58631?page=17),
[AM9 DV stalls](https://discourse.coreelec.org/t/ugoos-am9-pro-soc-s6-s905x5-j/58183?page=9), and
[FEL/DV failures](https://discourse.coreelec.org/t/fel-amlogic-no-support/58631?page=16).

The evidence therefore stays split. The new upstream removal of hard-coded QMS
may fix the perceived interpolation/cadence defect and must be tested first.
True DV still smearing at a verified fixed 60 Hz would remain a separate S6
target-DOVI pixel-path defect; it cannot be assigned to the C5 from public
evidence alone.
