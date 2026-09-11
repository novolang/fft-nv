# fft-nv

**Status: NOT IMPLEMENTED — interface only.**

Every public function below is published with its signature and its
effect row, and every body is `todo()`. Installing this package works;
calling it panics with `not implemented`.

## What this is

The fast Fourier transform, as a **plan built once per length** and run
against as many buffers as you have: radix-2 for a power of two,
mixed-radix over the factors for a composite length, and Bluestein's
chirp-z for a prime one — so every positive length transforms and the
plan tells you which of the three it chose and what it costs.

Around that: the real-input transform that answers the half spectrum
because the other half is a copy; 2-D transforms over ndarray-nv
matrices; the four window functions a spectrum needs, with both gain
corrections published because a tone and broadband noise need different
ones; the frequency axes, power spectrum, decibel conversion, shift and
peak finder a notebook reaches for; and a fixed-length module that
compiles for a microcontroller.

It is for the program that has samples and wants frequencies: a
spectrogram, an audio analyser, a convolution done the fast way, an
image filtered in the frequency domain, a sensor node watching for the
frequency a failing bearing rings at.

```
novo pkg add fft-nv
novo pkg build
novo test
```

## The one example that will work

```novo
use fftplan
use fftwin
use fftreal
use fftspec

// The spectrum of one window of a recording, in decibels.
fn spectrum_db(samples: [Float], rate: Float) -> Result<[Float], FftFault>
    let p = fftplan.of_len(samples.len())!
    let windowed = fftwin.apply(FftHann, samples, true)!
    let gain = fftwin.coherent_gain(FftHann, samples.len(), true)!
    let half = fftreal.forward(p, windowed, FftNormNone)!
    let power = fftspec.power(half, samples.len(), gain)!
    fftspec.to_db(power, 1.0)
```

Five calls, and four of them exist because the fifth would be wrong
without them: the window stops the seam of the buffer from smearing
energy across every bin, the gain undoes what the window took out,
`fftreal` computes half the bins because the other half is a mirror, and
the normalisation is named rather than assumed.

## The layer, and why

`core` — no effects at all.

A transform is arithmetic over numbers the caller already holds. A plan
is trigonometry over a length. Nothing is opened, nothing is waited for,
no clock is consulted, and the samples arrive as a buffer somebody else
read. The budget is `[]` on every one of the 69 public functions and
there was never any pressure on it.

**The device claim is built.** `tests/embedded_probe.nv` compiles
`fftfix` to a Cortex-M4 ELF for `--target=nrf52-qemu`. The consumer is a
condition-monitoring node: an accelerometer sampled at a kilohertz, a
64-point window filled one reading at a time, one forward transform, and
a look at which bin holds the energy — a bearing that has begun to fail
rings at a frequency a threshold can watch for. The whole pipeline is a
fixed-size buffer, six stages of butterflies and a comparison, and
nothing in it allocates.

## The load-bearing interface

```novo
pub struct FftPlan
pub fn of_len(points: Int) -> Result<FftPlan, FftFault> []
pub fn forward(p: FftPlan, buf: [Float], norm: FftNorm) -> Result<[Float], FftFault> []
```

**`FftPlan` is a value the caller builds once per length.**

An FFT of length n needs the n complex roots of unity and, for the
power-of-two case, the bit-reversal permutation that puts the input in
the order the butterflies read it. Both depend on n alone — not on the
data — so computing them once and reusing them across every window of a
stream is the entire difference between a transform that is affordable
in a loop and one that is not. A spectrogram of a ten-minute recording
runs one plan across six thousand windows.

rustfft calls this a plan too, and hands out a boxed trait object from a
planner that also caches. Here it is a plain **value**: `of_len` answers
an `FftPlan`, the caller keeps it, and every transform is a pure
function of it. It can sit in a struct beside the sample rate it belongs
to, be copied, be used from two places, and it cannot be stale, because
nothing mutates it. There is no planner and no cache — a value the
caller holds **is** the cache, and it is one they can see.

Three things follow from the plan being a value, and they are why it is
the load-bearing decision rather than an optimisation:

- **The algorithm choice is inspectable.** `algorithm(p)` says whether
  the length got radix-2, mixed-radix or Bluestein, and `inner_len(p)`
  says how big Bluestein's convolution is. A caller picking a window
  size can see that 1024 is cheap and 1021 carries a 2048-point
  convolution — *before* they build a pipeline around it.
- **Normalisation is NOT in the plan.** It is an argument, because the
  same plan serves both directions and a filter wants no scaling in the
  middle and one division at the end. Putting it in the plan would have
  made that caller build two.
- **The fallible half and the cheap half are separated.** `of_len` is
  where a length is checked; `forward` against a matching buffer cannot
  fail for any reason except the buffer's own length. A caller that
  builds one plan and runs ten thousand windows pays the check once.

## Two buffers, and which one each transform takes

The 1-D transforms take an **interleaved `[Float]`** —
`[re0, im0, re1, im1, …]`. The 2-D transforms take an **`NdFloat`** of
shape `[rows, cols, 2]`. That is not an inconsistency; it is the same
rule applied twice.

**A 1-D transform has no shape.** A complex sequence is a sequence. An
`NdFloat` is a buffer *plus* a shape, strides and an offset, so every
read through it is `data[offset + i * stride]` — and a transform's inner
loop does nothing but read, which puts the whole cost of the abstraction
exactly where this package cannot afford it. A rank-1 `NdFloat` with
stride 1 is the flat buffer with three extra numbers beside it, and the
API would spend every call checking that those three numbers are what it
needs.

**A 2-D transform IS a shape.** It is rows then columns: transform every
row, then transform every column of the result. The row and column
extents are not a convention the caller and this package agree on — they
are the operation. A flat list plus a row count would be this package
asking the caller to carry a shape beside a buffer, which is precisely
what `NdFloat` is, done badly.

And the two layouts are the same layout: `ndfloat.to_list` of an
`[r, c, 2]` array **is** the interleaved buffer. The third reason is
`fftfix`, which has to reach a device where an `NdFloat` never will —
keeping the 1-D surface on plain lists means the fixed-length module is
the same API at a different width rather than a second design.

## "In place" means value in, value out

rustfft's `process` takes `&mut [Complex]` and overwrites it. A `core`
package has no `[mutate]`, so every transform here takes a buffer and
answers a new one. Under Perceus a caller who writes

```novo
buf = fftcx.forward(p, buf, FftNormNone)!
```

and does not keep the old binding gets the reuse: the buffer is uniquely
referenced, so the new value is written into the old one's memory. That
is the in-place transform, spelled as a value. A caller who *does* keep
the old binding gets a copy, which is the honest cost of having asked
for both.

## What the device module does differently

`fftfix` is the same idea at a fixed width, and it differs from the host
modules in exactly two ways. Both are listed here because a reader
moving code between them should meet the differences in a table rather
than in a compiler error:

| | host (`fftcx`) | device (`fftfix`) |
| --- | --- | --- |
| buffer | interleaved `[Float]`, any length | `FftC64` / `FftC256`, a `@value` struct with inline arrays |
| length | the plan's, checked per call | the type's; nothing to check |
| normalisation | an `FftNorm` argument on every call | none — forward is unscaled, inverse carries the factor |
| windows | `fftwin`, which builds a coefficient list | none; a device compiles its own table |

The normalisation difference is the one worth arguing. A threshold that
compares bin powers does not care about a constant factor, and a scale
argument would be a branch inside the only loop on that path — so the
device transforms are unscaled forward, `1/N` inverse, and the round
trip is the identity. The host transforms take an `FftNorm` because a
notebook comparing a spectrum with numpy's genuinely needs the
convention named. `fftfix` also names **no type from any other module**,
and that is structural rather than stylistic: the device build compiles
that file and whatever its `use` lines reach, so a normalisation enum
borrowed from `fftplan` would have dragged `fftplan`'s `[Float]` fields
onto a target with no allocator.

Two widths and not a family, because a `@value` struct's array length is
part of its type and cannot be a parameter: every width is a *type* and
costs a full set of functions. 64 points is the fast loop; 256 is what a
vibration monitor wants. A third would be a third copy of everything for
a resolution somebody can get by decimating.

## The reference implementations, and what is specification

rustfft and `scipy.fft` are the references. The distinction matters
because it decides what a test may assert.

**Specification, and binding on this package**

- The DFT itself. For a given input and length there is one answer, and
  every algorithm here computes it: radix-2, mixed-radix and Bluestein
  agree, and the suite checks the cases whose answers are exact — a
  length-2 transform is one add and one subtract, a constant transforms
  to a single DC bin, an impulse transforms to a flat spectrum.
- **Conjugate symmetry of a real transform**, which is why `fftreal`
  answers `n/2 + 1` bins and why bin 0 and bin `n/2` are real.
- **The bin count is `n/2 + 1` and not `n/2`.** The plus one is the
  Nyquist bin. A 1024-point real transform gives 513 bins, and a caller
  who allocated 512 has written the bug `bin_count` exists to prevent.
- **`fftfreq`'s ordering**: positive frequencies then negative ones,
  which is the order the transform answers in and what numpy documents.
- **`fftshift` and `ifftshift` are different functions.** For an odd
  number of bins the two rotations differ by one, and a package with
  only one of them would round-trip wrongly for every odd length.
- The window formulas. Hann, Hamming and Blackman are three fixed sets
  of cosine coefficients, and Kaiser is the Bessel family they
  approximate.

**scipy's and rustfft's own choices, which this package follows and a
test may not treat as correctness**

- **Where the factor of n goes.** numpy and scipy scale the inverse;
  FFTW scales neither; a unitary transform splits it. All three are in
  use, so `FftNorm` names four conventions and **none of them is a
  default** — every transform takes one.
- **The Karatsuba-style thresholds**: when mixed-radix is preferred to
  Bluestein for a composite length with a large prime factor, and the
  radix a mixed-radix pass uses first. Speed decisions; the answers
  agree.
- **Rows before columns** in a 2-D transform. The two orders agree in
  exact arithmetic and differ in the last bits in floating point, so it
  is fixed here rather than left to an implementation — but it is a
  convention, not a theorem.
- **Symmetric vs periodic window sampling.** scipy's `get_window` is
  periodic and its `signal.windows.hann` is symmetric, which is a trap
  that has cost people real time. Here it is an explicit
  `periodic: Bool` on every call with no default, because there is no
  answer that is right for both callers.

So the correctness condition is **the exact cases are exact and the
identities hold**: forward then inverse is the identity, the real
transform agrees with the complex one over the first half, Parseval's
theorem relates the two energies, and the three algorithms agree with
each other at a length they all accept. A test that hard-coded
`numpy.fft.fft(x)` for an arbitrary `x` would be asserting this
implementation against a transcription.

## Deliberately not here

- **A planner with a cache.** rustfft has one because its plans are
  boxed trait objects a caller cannot easily hold. Here the plan is a
  value, so the caller's own binding is the cache — and one they can
  see the lifetime of.
- **The discrete cosine and sine transforms**, and the Hartley
  transform. Each is a real relative of the DFT with its own
  normalisation table and its own four variants, and folding them in
  here would double the surface for a different subject.
  `scipy.fft.dct` is one row; **a missing row**, named in this lane's
  report.
- **Convolution and correlation as named functions.** They are
  `forward`, `mul_spectra`, `inverse` — three calls a caller writes in
  a line — and wrapping them would mean choosing padding and edge
  behaviour on the caller's behalf. `mul_spectra` is here because the
  interleaved multiply is the loop an interleaved layout invites getting
  wrong.
- **The short-time Fourier transform and the spectrogram.** They are a
  windowing policy, an overlap and a matrix of results, which is a
  higher layer sitting on everything above; `scipy.signal` is where
  their neighbours are. **A missing row.**
- **Filter design.** A window function here is for spectral analysis;
  the same windows design FIR filters, and the rest of that subject —
  `firwin`, `freqz`, `filtfilt` — is a different package.
- **Single precision.** novo-lang's `Float` is a double, and a
  package that also carried an f32 path would be two implementations.
- **`@tier(embedded)` on anything but `fftfix`.** Every other module
  speaks `[Float]`, `[Int]`, `Str` or `NdFloat`, and a list is a heap
  allocation the tier refuses.

## Status

Every function is `todo()`. `novo test` runs the API suite, and every
assertion in it reaches `not implemented: fft-nv.<module>.<fn>` — which
is the expected result until the bodies land, and is what makes the
suite a description of the interface rather than of nothing.
`novo test --isolate` is the readable form: one verdict per test, naming
the function it stopped at.

| module | public types | functions | constants | implemented |
| --- | --- | --- | --- | --- |
| `fft2d` | 0 | 10 | 0 | no |
| `fftcx` | 0 | 10 | 0 | no |
| `fftfault` | 1 | 1 | 0 | no |
| `fftfix` | 2 | 18 | 2 | no |
| `fftplan` | 3 | 9 | 0 | no |
| `fftreal` | 0 | 6 | 0 | no |
| `fftspec` | 0 | 10 | 0 | no |
| `fftwin` | 1 | 5 | 0 | no |
| **total** | **7** | **69** | **2** | **no** |
