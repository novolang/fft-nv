# fft-nv

The discrete Fourier transform turns a sequence of samples into the
amplitudes of the frequencies that make it up. The fast Fourier
transform, or FFT, is an algorithm that computes it in `n log n` steps
instead of `n²`. This package brings it to novo-lang, with the window
functions and spectrum arithmetic that go with it. Its references are
the Rust crate [rustfft](https://docs.rs/rustfft) and
[`scipy.fft`](https://docs.scipy.org/doc/scipy/reference/fft.html). Its
two-dimensional transforms read matrices from
[ndarray-nv](https://novo-lang.org/packages/ndarray-nv).

**Status: NOT IMPLEMENTED — interface only.** Every function is declared
with its full signature, but every body is a `todo()` that panics when
called. The package is published so its design can be reviewed and
depended on before it is implemented. Version 0.1.0 will be the first
working release.

## What the transform answers

A transform of `n` samples answers `n` **bins**. Bin `k` holds a complex
number whose size is how much of frequency `k` the signal contains and
whose angle is that frequency's phase. Frequency `k` is `k` cycles
across the whole buffer, so with a sample rate the bins become hertz.

A **complex number** is held here as two floats side by side, the real
part then the imaginary part. A buffer of `n` complex numbers is
therefore a list of `2n` floats, called an **interleaved** buffer.

The **inverse transform** turns bins back into samples. Forward followed
by inverse is the original signal, up to a factor of `n` that somebody
has to divide out. Which of the two directions carries that factor is a
**normalisation** convention, and three different ones are in common
use, so this package names all four and defaults to none.

A **real** input, which is what a microphone or an accelerometer gives,
has a spectrum whose second half is the mirror image of its first. The
real transform therefore answers `n/2 + 1` bins and not `n`. The extra
one is the **Nyquist bin**, at half the sample rate.

A **window function** tapers a buffer towards zero at both ends. Without
one, the join between the end of the buffer and the beginning of the
next is a sharp step, and a sharp step spreads energy across every bin.
A window removes some of the signal's energy, so a measurement made
through one is divided by a **gain** to put it back. There are two
gains: a **coherent** gain for a pure tone and a **noise** gain for
broadband noise.

A **plan** is the trigonometry a transform of one particular length
needs: the roots of unity, and the permutation that puts the input in
the order the algorithm reads it. It depends on the length alone and not
on the data, so it is built once and used for every buffer of that
length. A spectrogram of a ten-minute recording runs one plan across
thousands of windows.

Three algorithms cover every length, and a plan reports which one it
chose.

| Length | Algorithm | Cost |
| --- | --- | --- |
| a power of two | radix-2 | the cheapest case |
| composite | mixed-radix over the factors | close to radix-2 |
| prime | Bluestein's chirp-z | a convolution of a larger, power-of-two length |

## Install

```
novo pkg add fft-nv
```

## Example

```novo
use std.list
use fftplan
use fftwin
use fftreal
use fftspec

fn main() [io]
    // Eight samples, standing in for one window of a recording.
    let samples = [0.0, 0.7, 1.0, 0.7, 0.0, -0.7, -1.0, -0.7]

    // The plan holds the roots of unity for this length. Build it once
    // and run it against every window of the recording.
    match fftplan.of_len(8)
        Err(e) => println(e.message())
        Ok(p) =>
            // The window tapers the ends, so the seam between one buffer
            // and the next does not smear energy across every bin.
            // `true` asks for periodic sampling rather than symmetric.
            match fftwin.apply(FftHann, samples, true)
                Err(e) => println(e.message())
                Ok(windowed) =>
                    // A real input has a mirrored spectrum, so this answers
                    // five bins for eight samples and not eight.
                    match fftreal.forward(p, windowed, FftNormNone)
                        Err(e) => println(e.message())
                        Ok(half) =>
                            // One magnitude per bin.
                            match fftspec.magnitude(half)
                                Err(e)    => println(e.message())
                                Ok(mags)  => println("${list.len(mags)} bins, and bin_count says ${fftreal.bin_count(8)}")
```

Build and test with `novo pkg build` and `novo test`. Today `novo test`
fails on purpose: every test reaches a
`not implemented: fft-nv.<module>.<fn>` panic. The tests are the
specification the implementation will have to satisfy.

## What the package contains

| Module | Contents |
| --- | --- |
| `fftplan` | The plan, the four normalisation conventions, the three algorithms, and the length arithmetic: is this a power of two, what is the next one, what are the factors. |
| `fftcx` | The one-dimensional complex transforms over an interleaved buffer, the direct radix-2 forms, the conversions between separate and interleaved parts, the spectrum product and the conjugate. |
| `fftreal` | The real-input transform and its inverse, the bin count, the frequency axis for it, and the conversions between a half spectrum and a full one. |
| `fftspec` | What a caller does with a spectrum: magnitude, squared magnitude, phase, power with a window gain divided out, decibels, the two shifts, the frequency axis, and the peak. |
| `fftwin` | Five window functions, their coefficients, applying one to a buffer, and the two gain corrections. |
| `fft2d` | The two-dimensional transforms over an ndarray-nv matrix, along both axes or along one, with the magnitude, the two shifts and the shape helpers. |
| `fftfix` | Fixed-length 64-point and 256-point transforms over a value type with inline arrays, which build for a microcontroller. |
| `fftfault` | Every reason a transform refuses, as one enum with eleven variants. |

## How to choose an entry point

**Build a plan with `fftplan.of_len`, then use `fftcx` for complex input
and `fftreal` for real input.** A real input through `fftcx` costs twice
the work and answers a spectrum whose second half you already knew.

**Use `fftplan.algorithm` before you fix a window size.** A length of
1024 gets radix-2. A length of 1021 is prime and gets Bluestein, which
runs a 2048-point convolution underneath. `fftplan.inner_len` says how
big that convolution is.

**Use `fft2d` for an image or any matrix.** It takes an ndarray-nv array
of shape rows by columns by 2, transforms every row and then every
column, and needs one plan per axis.

**Use `fftfix` on a microcontroller.** See "Running on a
microcontroller".

**The one-dimensional transforms take an interleaved list and the
two-dimensional ones take a shaped array.** A complex sequence is a
sequence, and reading it through a shape and a stride would put an
indirection inside the only loop that matters. A two-dimensional
transform, by contrast, is rows and then columns, so the extents are the
operation rather than a convention. The two layouts hold the same bytes:
`ndfloat.to_list` of a rows-by-columns-by-2 array is the interleaved
buffer.

## The rules a user needs

1. **Every transform takes a normalisation and none is the default.**
   `FftNormNone` scales neither direction, which is FFTW's convention.
   `FftNormBackward` scales the inverse by one over `n`, which is what
   NumPy and scipy do. `FftNormForward` scales the forward direction.
   `FftNormOrtho` gives each direction one over the square root of `n`.
   A filter wants no scaling in the middle and one division at the end,
   which is why the choice is an argument and not part of the plan.
2. **A real transform answers `n/2 + 1` bins, not `n/2`.** A 1024-point
   real transform gives 513 bins. `fftreal.bin_count` is that number. Bin
   0 and the Nyquist bin are real.
3. **Every window call takes `periodic` and none is the default.**
   scipy's `get_window` samples a window periodically and its
   `signal.windows.hann` samples it symmetrically, and the two differ.
   Spectral analysis wants the periodic form. Filter design wants the
   symmetric one.
4. **Divide a windowed measurement by the right gain.** Use
   `fftwin.coherent_gain` when you are measuring the amplitude of a
   tone, and `fftwin.noise_gain` when you are measuring the power of
   broadband noise. `fftspec.power` takes the gain as an argument for
   this reason.
5. **`fftshift` and `ifftshift` are different functions.** They rotate a
   spectrum so that frequency zero is in the middle, and back. For an
   odd bin count the two rotations differ by one place, so using one for
   both does not round-trip.
6. **`fftspec.fftfreq` lists the positive frequencies first and then the
   negative ones**, which is the order the transform answers in, and
   what NumPy documents. `fftreal.rfftfreq` is the axis for a half
   spectrum.
7. **A two-dimensional transform does rows and then columns.** The two
   orders agree in exact arithmetic and differ in the last bits in
   floating point, so the order is fixed here rather than left to an
   implementation.
8. **Nothing is transformed in place.** Every function takes a buffer
   and answers a new one. When the caller does not keep the old binding,
   as in `buf = fftcx.forward(p, buf, FftNormNone)!`, the old buffer is
   uniquely referenced and the new value is written into its memory. A
   caller who does keep the old binding gets a copy, which is what
   keeping both costs.
9. **An interleaved buffer has an even number of floats.** An odd count
   is `FftOddInterleave`.
10. **Convolution is three calls.** Transform both inputs, multiply the
    spectra with `fftcx.mul_spectra`, and transform back. There is no
    `convolve`, because it would have to choose the padding and the edge
    behaviour for you.

## Running on a microcontroller

novo-lang lets a package state which of its modules can run on a device
with no heap allocator, and the compiler checks that claim on every
build. Here the claim covers `fftfix` alone.

`fftfix` holds a buffer in a value type with an inline array, at one of
two fixed lengths: `FftC64` for 64 points and `FftC256` for 256. The
length is part of the type, so nothing is checked per call and nothing
is allocated from start to end.

```bash
novo build --target=nrf52-qemu tests/embedded_probe.nv
```

That command builds a Cortex-M4 executable today. The probe is a
condition-monitoring node: an accelerometer read at a kilohertz, a
64-point buffer filled one sample at a time, one forward transform, and
a look at which bin holds the energy. A failing bearing rings at a
frequency a threshold can watch for.

The probe builds; it does not run. Every function it calls is a `todo()`
today, so a device that executed it would panic in the first line of its
main function. What the build checks is that the compiler accepts every
one of these functions for a device: no list literal, no string
concatenation, no unbounded loop.

**A device cannot depend on this package as a whole.** The other seven
modules speak lists, strings and ndarray-nv arrays, and one host-only
function anywhere in a compilation unit is an undefined symbol at link
time on a device, whether or not the firmware calls it. The probe is
built against `fftfix` alone, and `fftfix` names no type from any other
module here.

`fftfix` differs from the host modules in two ways, and both are on
purpose.

| | `fftcx`, on a host | `fftfix`, on a device |
| --- | --- | --- |
| Buffer | an interleaved list, any length | `FftC64` or `FftC256`, a value type with inline arrays |
| Length | the plan's, checked on every call | the type's, so nothing is checked |
| Normalisation | an argument on every call | none: forward is unscaled and the inverse carries the factor |
| Windows | `fftwin` builds a coefficient list | none; a device compiles its own table |

A threshold that compares bin powers does not care about a constant
factor, and a scale argument would be a branch inside the only loop on
that path. The device round trip is still the identity.

There are two widths rather than a family because the length of a value
type's inline array is part of its type and cannot be a parameter. Each
width costs a full set of functions. Sixty-four points is the fast loop
and 256 is what a vibration monitor wants.

## What is not included

- **A planner with a cache.** The plan is a value the caller holds, so
  the caller's own binding is the cache.
- **The discrete cosine and sine transforms, and the Hartley
  transform.** Each is a real relative of the discrete Fourier transform
  with its own four variants and its own normalisation table.
- **Named convolution and correlation functions.** See rule 10.
- **The short-time Fourier transform, and the spectrogram.** They are a
  windowing policy, an overlap and a matrix of results, which sits above
  everything here.
- **Filter design.** The windows here are for spectral analysis. Filter
  design, frequency response and zero-phase filtering are a different
  subject.
- **Single precision.** novo-lang's `Float` is a double, and a second
  path at single precision would be a second implementation.
- **A device build of anything but `fftfix`.** See "Running on a
  microcontroller".

## Related packages

- [ndarray-nv](https://novo-lang.org/packages/ndarray-nv) is the matrix
  the two-dimensional transforms read. `fftfix` uses none of it, which
  is what lets the device probe reach that module alone.
- [stats-nv](https://novo-lang.org/packages/stats-nv) summarises the
  samples that go in and the powers that come out.
- [plot-nv](https://novo-lang.org/packages/plot-nv) draws a spectrum.
- [image-nv](https://novo-lang.org/packages/image-nv) holds the images
  that `fft2d` filters in the frequency domain.

## Tests

```bash
novo test tests/fft_tests.nv          # 34 tests: the exact cases, the identities, the windows
novo build --target=nrf52-qemu tests/embedded_probe.nv
```

A transform of arbitrary data has no closed form, so a suite that
recorded `numpy.fft.fft(x)` for a random `x` would be asserting this
implementation against a transcription. The suite asserts three kinds of
thing instead.

The first is the cases whose answers are exact. A length-2 transform is
one addition and one subtraction. A constant signal transforms to a
single bin at zero frequency. A single non-zero sample transforms to a
flat spectrum. A pure tone at exactly one bin's frequency transforms to
two bins.

The second is the identities, which hold for any input: forward then
inverse is the original, Parseval's theorem relates the energy in the
samples to the energy in the bins, the real transform agrees with the
complex one over the first half, `fftshift` and `ifftshift` invert each
other, and the three algorithms agree at a length all of them accept.

The third is the window coefficients, which scipy prints and which are
three fixed sets of cosine terms plus the Bessel family that Kaiser
approximates.

`tests/embedded_probe.nv` is that claim written as a program. The command
above builds a Cortex-M4 executable today.

The tests compile today and fail at run, each on the
`not implemented: fft-nv.<module>.<fn>` panic that is its body. That is
the expected state of an interface release. They turn green one at a
time as bodies land.

## Implementation status

| Item | Implemented |
| --- | --- |
| `fftfix.FFTFIX_C64_POINTS`, `.FFTFIX_C256_POINTS` | yes (they are constants) |
| `fftplan.FftPlan`, `.FftNorm`, `.FftAlgo`, `fftwin.FftWindow` | declared |
| `fftfix.FftC64`, `.FftC256`, `fftfault.FftFault` | declared |
| `fftplan.of_len`, `.len`, `.float_len`, `.algorithm`, `.inner_len` | no |
| `fftplan.is_power_of_two`, `.next_power_of_two`, `.factors`, `.scale` | no |
| `fftcx.forward`, `.inverse`, `.forward_radix2`, `.inverse_radix2` | no |
| `fftcx.interleave`, `.real_parts`, `.imag_parts`, `.of_real`, `.mul_spectra`, `.conjugate` | no |
| `fftreal.bin_count`, `.forward`, `.inverse`, `.rfftfreq`, `.to_full`, `.to_half` | no |
| `fftspec.magnitude`, `.magnitude_squared`, `.phase`, `.power`, `.to_db` | no |
| `fftspec.fftshift`, `.ifftshift`, `.fftfreq`, `.peak_bin`, `.peak_frequency` | no |
| `fftwin.coefficients`, `.apply`, `.coherent_gain`, `.noise_gain`, `.name` | no |
| `fft2d.of_real`, `.real_part`, `.forward`, `.inverse`, `.forward_axis`, `.inverse_axis` | no |
| `fft2d.magnitude`, `.fftshift`, `.ifftshift`, `.complex_shape` | no |
| `fftfix`'s eighteen functions over the two fixed widths | no |
| `fftfault`'s eleven variants, `.is_shape_fault` and its `Error` implementation | no |

## Licence

Apache-2.0. See `LICENSE`.

<!-- docs/writing-a-readme.md is the style guide for this page. -->
