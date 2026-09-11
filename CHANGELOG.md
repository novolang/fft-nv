# Changelog

Every published version, newest first. This file is on the publish
allow-list, so it travels with the package: it is the only thing a
consumer deciding whether to upgrade can read.

## 0.0.1 — 2026-09-11

The **interface**, before anyone implements it.  Every signature, every
type and every effect row is published; every body is `todo()`, and the
release is stamped `NOT IMPLEMENTED — interface only`.  Adding this
package works and calling it panics.

- Eight modules.  `fftplan` is the plan and the normalisation
  conventions; `fftcx` is the complex transform; `fftreal` is the
  real-input transform and its half spectrum; `fftwin` is the windows;
  `fftspec` is what a notebook does with a spectrum afterwards; `fft2d`
  is two dimensions over ndarray-nv; `fftfix` is the fixed-width
  transform a device uses; `fftfault` is every refusal.
- **`FftPlan` is the load-bearing interface** — a value built once per
  length and run against as many buffers as a stream has.  rustfft
  hands out a boxed trait object from a caching planner; here the plan
  is a value the caller holds, so their own binding is the cache and
  they can see its lifetime.  The algorithm choice is inspectable
  (`algorithm`, `inner_len`), so a caller picking a window size can see
  that 1024 is radix-2 and 1021 carries a 2048-point convolution before
  they build a pipeline around it.
- **Normalisation is an argument, not a plan field**, and there is no
  default: numpy scales the inverse, FFTW scales neither, a unitary
  transform splits it, and all four conventions are named in `FftNorm`.
  The same plan serves both directions, and a filter wants no scaling
  in the middle.
- **The 1-D transforms take an interleaved `[Float]` and the 2-D ones
  take an `NdFloat`**, and the README argues why that is one rule
  applied twice: a 1-D transform has no shape and would pay for
  `NdFloat`'s strides on every read, while a 2-D transform IS rows then
  columns and the shape is the operation.  The two layouts are the same
  bytes — `to_list` of an `[r, c, 2]` array is the interleaved buffer.
- **"In place" is value in, value out.**  A `core` package has no
  `[mutate]`, so `buf = fftcx.forward(p, buf, norm)!` with no other
  binding live is the Perceus reuse, which is the in-place transform
  spelled as a value.
- **The device claim is built.**  `tests/embedded_probe.nv` compiles
  `fftfix` to a Cortex-M4 ELF for `--target=nrf52-qemu`: a
  condition-monitoring node filling a 64-point window one ADC reading
  at a time, transforming it, and looking at which bin holds the
  energy.  `fftfix` names no type from any other module, because the
  device build compiles that file and whatever its `use` lines reach —
  so it takes no `FftNorm`, is unscaled forward and carries the factor
  in the inverse.
- **Both window gain corrections are published.**  A tone's amplitude
  is corrected by the sum of the coefficients and broadband noise by the
  sum of their squares; a package offering one of them would be wrong
  for half its callers without saying so.  `coherent_gain` and
  `noise_gain` are named for what they correct.
- **Symmetric or periodic is an explicit argument with no default**,
  because scipy's `get_window` is periodic and its
  `signal.windows.hann` is symmetric, and the two differ by one sample
  in every overlap-add reconstruction.
- **The real transform answers `n/2 + 1` bins and the plus one is
  Nyquist.**  `bin_count` exists so that nobody allocates 512 for a
  1024-point transform.
- Every vector is an exact case (a length-2 transform, a constant, an
  impulse), an identity that holds for any input (the round trip,
  Parseval, the half against the full), or a published number (the
  window formulas, numpy's frequency axes).
