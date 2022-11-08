+++
draft = true
date = 2022-10-18T00:28:04+02:00
title = "DSP with Zig - Oscillators (Part 1)"
description = "Making blips and blops in Zig"
slug = "dsp-with-zig-oscillators"
authors = ["Riccardo Binetti"]
tags = ["zig", "dsp", "oscillators", "blit", "band limited oscillators", "band limited impulse train"]
categories = ["dsp", "zig"]
externalLink = ""
series = ["DSP with Zig"]
+++
During my recent visit at [SYCL](https://sycl.it) I had the occasion to talk about, among other
things, digital audio programming with many people. This fueled back my desire to start digging into
DSP and write some usable DSP code (possibly to run it on some [specialized
hardware](https://github.com/rbino/zigthesizer)) in the future).

Hence, I decided to start a series of blogposts that will explore various aspects of digital
synthesis and processing. Since often DSP code tends to be quite dense, both for performance reasons
and because it uses variable names directly derived from compact mathematical formulae, I will also
try to focus on the readability of the code as much as possible. This also means that I will accept
some code duplication instead of, e.g., using the Zig interfaces pattern, since my hope is that this
remains clear enough to be used as reference even by people with little or no Zig experience.

This series is also first and foremost my personal learning gym for both DSP and Zig, so I gladly
accept feedback, suggestions, fixes etc. I will paste snippets of code directly in the posts, but I
will also commit the full working code into a [companion
repository](https://github.com/rbino/zig-blips) so that it can be easily tested (and heard).

In the first part of the series, I will focus on oscillators, since they are the basic building
block to emit sounds in the digital synthesis world.

## Oscillators

An oscillator can be defined as some kind of object that produces a periodic waveform. Oscillators
exist also in the physical realm (both mechanical and electronic), but in our case we're going to
focus on digital oscillators, so our "object" will actually be some code.

The produced periodic waveform can be arbitrary, but at least in this post we're going to
concentrate on producing so-called Virtual Analog waveforms, which are the most common waveforms
found in analog synths: sine, saw, square and triangle.

Each oscillator behaves differently, but let's explore some things that more or less all oscillators
will have so that we can have some background before diving into the implementation.

### Initialization

The oscillator structs will have an `init` function which takes some parameters and returns an
initialized oscillator. All oscillator will accept at least the initial frequency as a parameter,
some oscillators might need additional parameters.

### Frequency change

The oscillator must have a way to change its frequency, so the oscillator struct will have a
`setFrequency` method. This is useful (opposed to, e.g., directly changing the `frequency` field)
because many oscillators have parameters that are fixed given a fixed frequency. This allows to
recalculate them only once when the frequency is changed.

### Phase update and output

All oscillators must store the information that indicates at what point of the waveform they're at.
This is called _phase_. Since the waveform is periodic, the phase usually "wraps around" at the end
of the cycle of the waveform, because the value of the waveform at phase `x` and at phase `x +
oneFullCycle` is the same.

There are different ways to store and increment the phase. Some code directly uses the phase value
that will be fed to the oscillator function (for example, the phase for a sine oscillator could
start from `0` and max out at `2 * pi`, wrapping back to `0`).

In my examples, unless otherwise specified, the `phase` variable contains a floating point value
that goes from `0.0` to `1.0`. This makes the code to update the phase uniform, namely:

```zig
// delta_time is the time between two consecutive samples
// const delta_time = 1 / sample_rate;
// sample_rate is the system sample rate, i.e. usually 44_100 or 48_000 Hz
pub fn tick(self: *Self, delta_time: f32) void {
    // Compute the output sample before updating the phase
    // ...
    // const out = ...

    // self.freq is the frequency of the oscillator in Hertz, e.g. 110
    const delta_phase = delta_time * freq;
    self.phase += delta_phase;
    // Wrap around phase
    self.phase -= @floor(self.phase);

    return out;
}
```

The last line is just a shortcut to avoid an `if` and is basically equivalent to

```zig
if (self.phase >= 1.0) {
  self.phase -= 1.0;
}
```

My assumption is that avoiding a conditional makes the code faster (full disclosure: I didn't
benchmark it, I just saw that all DSP code usually does it that way), and I think it's readable
enough to leave it as it is.

The other role of the `tick` function, as shown above, is to produce an output sample for the
current phase value. The output sample will be a float value in the range `-1.0` to `1.0`.

Now let's analyze the naive implementations of the oscillators for the waveforms we mentioned early.
We will see why they're naive in a little bit, let's start with the sine wave.

### Sine

TODO: sine image

The sine wave is the fundamental waveform when it comes to synthesis. This is because it is
the most pure wave, since its frequency spectrum only contains its root frequency.

Here you can see the frequency spectrum[^1] of a sine wave with a frequency of 110Hz, and as you can
see the only frequency contained in the signal is, indeed, 110Hz.

TODO: 110Hz sine FFT

The 

[^1]: In this post I won't go into much detail about frequency spectra and Fourier Transforms, the
tl;dr version of what this graphs show is "all the frequencies contained in a sound", head over to
[Wikipedia](https://en.wikipedia.org/wiki/Fourier_transform) for further
details.
