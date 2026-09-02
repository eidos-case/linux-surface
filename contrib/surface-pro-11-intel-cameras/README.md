# Surface Pro 11 (Intel) — libcamera tuning for the three cameras

The Surface Pro 11 with Intel (Lunar Lake) has three cameras: a 13 MP rear
`ov13858`, a 5 MP front `imx681`, and an infrared `vd55g0` used for face login.
With current kernels and libcamera they enumerate and stream, and the picture is
unusable — wrong colour, wrong exposure, or both. What is missing is tuning
data, which is per-sensor and per-board and cannot come from upstream.

These three files are that data, measured on one machine against the same scenes
shot under Windows. **Treat them as a starting point**: they encode this unit's
sensor and lens, and yours may differ enough to want re-measuring.

## Where the files go

    /usr/share/libcamera/ipa/simple/ov13858.yaml
    /usr/share/libcamera/ipa/simple/imx681.yaml
    /usr/share/libcamera/ipa/simple/vd55g0.yaml

The directory is where your distribution installs libcamera's soft-ISP tuning;
on Ubuntu 26.04 that is the path above. libcamera picks the file by sensor name,
so no configuration is needed beyond putting them there.

## What you also need, and do not have yet

These files alone are not enough. As of 2026-09-03:

- **The rear camera needs a kernel patch.** `ov13858` assumes it is already
  powered at probe, which is true where ACPI power resources do the work and
  false here, where an INT3472 companion registers regulators, a clock and a
  reset GPIO for the driver to consume. Without it: `failed to find sensor: -5`.
  Sent to linux-media on 2026-08-31, not merged.

- **The front camera needs a libcamera patch.** `imx681` has no
  `CameraSensorHelper`, so libcamera cannot convert its gain code and the picture
  comes out about 4.5x under-exposed. One small class, sent to libcamera-devel on
  2026-08-31, not merged. Until it is, `imx681.yaml` will not help on its own.

Check whether these have landed before assuming the files are at fault.

## Things that will look like bugs and are not

**`imx681.yaml` has no `Ccm` block.** That is deliberate. With a colour matrix
this sensor turns clipped highlights magenta; without one the colour is slightly
flat and always sane. If you add a matrix, check a scene with a bright window in
it before deciding you have improved anything.

**`vd55g0.yaml` has no `Agc` block.** Also deliberate: the infrared camera is
paired with an illuminator, and letting the AGC hunt makes face recognition less
reliable, not more.

**These files are libcamera-version-specific.** They are written for 0.7.0. The
same rear matrix on libcamera master gives a neutral-grey channel spread of 13.9
where 0.7.0 gives 1.06 — the pipeline changed underneath. If you move to a newer
libcamera, re-measure rather than carrying these across.

## Face login

`howdy/libcamera_reader.py` is a Howdy recorder plugin that reads the infrared
camera through libcamera's Python bindings, since Howdy expects a V4L2 device
and the IR camera is only usable through libcamera here. Point Howdy's
`device_path` at it per its own documentation.

Two things in it are load-bearing and easy to lose when adapting it:

- `get()` must return the real frame dimensions. Returning zeros makes Howdy's
  comparison step request an enormous resize — it asked for 118 GB here before
  it was fixed.
- `release()` must be guarded against being called twice. libcamera aborts
  inside `PipelineHandler::stop()` on the second call, taking the process with
  it.

## Provenance

Measured on a Surface Pro for Business 11th Edition with Intel, SKU
`Surface_Pro_11th_Edition_With_Intel_For_Business_2103`, Core Ultra 7 268V,
Ubuntu 26.04, kernel 7.0.0-30, libcamera 0.7.0. The rear colour matrix and gain
curve were derived by shooting the same scene under Windows and under Linux
minutes apart and comparing neutral surfaces, not by eye.
