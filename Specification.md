# Music Optimized for Paula (MOP) Specification

_Marshall Elfstrand, Spring 2025_


## Container Formats

MOP data is stored in the
[Interchange File Format](https://en.wikipedia.org/wiki/Interchange_File_Format)
(IFF) commonly used on the Amiga, which was introduced by Electronic Arts in
1985. An IFF file is composed of "chunks", where each chunk has a four-character
ID, followed by a 32-bit big-endian unsigned byte count, followed by the chunk data.
Chunks may be nested, with the root chunk being of type `FORM` (the most
common), `LIST`, or `CAT`.

The MOP song data is stored as a `FORM` of type `MOP0`. This can be the sole
contents of a file (typically with a `.mop` extension), or it can be embedded
alongside samples in a `CAT` of type `MOPS` (typically with a `.mops` extension,
for "MOP + Samples").

### `FORM` `MOP0`

A `FORM` `MOP0` supports the standard IFF chunks such as `NAME` (song name),
`AUTH` (author), `(c)` (copyright), `ANNO` (annotations/notes), etc.
The chunk types specific to `MOP0` are `CHB0`/`CHW0`, `CHB1`/`CHW1`,
`CHB2`/`CHW2`, `CHB3`/`CHW3`, `META`, and `INSM`. All chunks are optional,
though of course there wouldn't be much of a point to not have at least one
chunk of channel data.

The `CHBx` and `CHWx` chunks map to the four Amiga hardware sound channels
(for example, `CHB0` and `CHW0` are for hardware channel 0, the first left
channel), and contain streams of music playback events and their parameters,
described below.

Metadata events are stored in the `META` chunk. This is a single stream of
32-bit values containing metadata events and their parameters, described below.

The `INSM` chunk provides an optional "instrument map" that tells the player
how to interpret the instrument numbers used in the data. Each instrument can
be mapped to an internal sample index (embedded in a MOPS file) or an external
sample ID (for lookup in a shared sound bank for e.g. a game). If omitted, the
instrument numbers are all mapped directly to the indices of the samples
embedded in a MOPS file.

### `CAT` `MOPS`

MOP data may be grouped with its samples in an IFF `CAT` (concatenation) chunk
of type `MOPS`. This is expected to contain at least two chunks: a `FORM` `MOP0`
for the song data, and a `LIST` `8SVX` for the sound samples referenced by
the song.


## Channel Event Streams

The Motorola 68000 can only load 16-bit and 32-bit values from even-numbered
memory addresses. To support this, channel event data is split into streams
of byte- and word-sized values: that is, unsigned 8-bit and 16-bit values.
Operations, deltas, instrument numbers, and volume levels are stored in the
byte stream (`CHBx`), while sample periods are stored in the word stream
(`CHWx`). If there are an odd number of bytes in the byte stream, that will
be handled by the padding inherent to IFF chunks.

Each event begins with a single byte in the byte stream that indicates both
the time delta — that is, the number of ticks to wait — and the operation to
perform. The lower 5 bits are the delta, and the upper 3 bits are the operation.

```
%oooddddd
```

When reading an event, the player starts a countdown from the delta value, and
when it reaches zero, it performs the operation. A delta of zero means to
perform the operation immediately and move to the next event. The maximum delta
value is 31, which is just over half a second at 60 ticks per second; further
delays are accomplished by leaving the operation at 0 (delay only) and using
additonal delay events as needed.

The supported operations and parameters in `MOP0` are:

| Dec | Bin | Description                                 |
|-----|-----|---------------------------------------------|
| 0   | 000 | No operation (delay only)                   |
| 1   | 001 | Note off                                    |
| 2   | 010 | Modify period: period                       |
| 3   | 011 | Play sample with period: period, instrument |
| 4   | 100 | Modify volume: volume                       |
| 5   | 101 | Play sample with volume: volume, instrument |
| 6   | 110 | Modify volume and period: volume, period    |
| 7   | 111 | Play sample: volume, period, instrument     |

Operations that play a sample start the sample from the beginning, and any
values that are not specified preserve the existing value on the channel.
For example, operation 3 starts playback of a sample with a given sample
period at the existing volume on the channel.

The operation bit flags are ordered so that the highest bit indicates that a
volume value should be read from the byte stream, and the second highest
bit indicates that a sample period value should be read from the word stream
(most sigificant byte first, i.e. big-endian). The third highest bit requires
an added check to see if either a volume or sample period is present to
determine whether to read an instrument number value from the byte stream or
whether to silence what is playing on the channel. Note that the volume value
for an event is stored _before_ the instrument number in the byte stream.

Instrument numbers may be any value from 0 to 255. Volume levels may be any
value from 0 to 64 (yes, including 64). Sample periods may technically be any
value from 0 to 65,535, but the Amiga hardware doesn't support sample periods
lower than around 124 (28 kHz), and even the extended octaves used by some MOD
players top out at around 1712. See below for an introduction to sample periods.


## Metadata Event Streams

The metadata channel is for events that aren't related to sounds playing on
a specific hardware channel, such as setting loop points, triggering events
in an animated sequence in a game, changing the playback speed, and so on.

Metadata events are always four bytes: a 16-bit delta value, an 8-bit operation,
and an 8-bit argument. Some operations have no argument, in which case a 0 is
used as the argument byte.

The supported metadata operations in `MOP0` are:

* `0`: No operation (delay only). The argument is ignored.
* `1`: Set timing. The argument is the number of ticks per second.
  By default, MOP data plays back at 60 ticks per second, reflecting its
  MIDI influence.
* `2`: Set loop point. When the end of all channel data has been reached
  (including the metadata channel), the song will loop back to here.
* `3`: Set cue. The argument is the cue number (0 to 255). This is reported
  by the player code to trigger custom behavior in e.g. a game/demo.
* `4`: Set preferred FX channel. The argument is a channel number (0 to 3)
  or 255 to indicate "no preference" (the default). This is used to suggest
  to the player which channel is best to temporarily replace with sound effects
  if needed.


## The Instrument Map

The optional `INSM` chunk is a sequence of 16-byte structures that indicate
which sample to use for a given instrument number in the channel data and
how to play and loop it. The instrument number used in the channel data refers
to the 0-based index of the entry in the instrument map.

```
UWORD sampleNumber
UWORD flags
ULONG initialWordOffset
UWORD initialWordCount
ULONG repeatWordOffset
UWORD repeatWordCount
```

The low bit of the `flags` indicates whether the sample is internal (0) or
external (1). Internal sample numbers are treated as the 0-based index of
the sample in the `LIST` `8SVX` of a `CAT` `MOPS` container. External sample
numbers are an identifier between 0 and 65,535 to be interpreted by the player
(for example, referencing game resource IDs). Currently this is the only flag
bit in use.

The `initialWordOffset`, `initialWordCount`, `repeatWordOffset`, and
`repeatWordCount` indicate the point at which to start playing the sample,
the amount of data to play before looping, the point at which to play the
repeat portion of the sample, and amount of data to play for the repeat
portion, respectively.

- **Important:** These values are expressed as counts of 16-bit words, not of bytes.
  For example, if you want to start playback 300 bytes into a sample, you'd set
  the `initialOffsetWord` to 150.

If the `INSM` chunk is not present, the instrument numbers in the channel data
are all assumed to be 0-based indices of samples in the `LIST` `8SVX` of the
`CAT` `MOPS` container, and the `oneShotHiSamples` and `repeatHiSamples` in
the `8SVX` data are used to determine how to loop the sample.


## About Sample Periods

The playback rate for sound on the Amiga is specified using a "sample period",
which is the number of system clock ticks that the Paula sound chip should spend
playing each individual 8-bit sample value in the sampled sound data.

These "ticks" are not the same as the music playback "ticks" (which are
typically 50 or 60 ticks per second); instead, they're derived from the speed
of the custom chips used in the original Amiga 1000, 500, and 2000, which
was closely tied to the speed of the Motorola 68000 CPU used in those systems.

This speed varied between NTSC (North American) and PAL (European) systems:

| Standard | CPU Speed | Ticks/s   |
|----------|-----------|-----------|
| NTSC     | 7.159 MHz | 3,579,545 |
| PAL      | 7.094 MHz | 3,546,895 |

If you want to play back a sound sample at 8,192 Hz on an NTSC Amiga, for
example, you divide 3,579,545 by 8,192, resulting in 437 for the sample period.
This is the value that gets sent to the Paula chip to tell it how fast to play
back the sample.

This dependency on the region-specific custom chip speed means that samples
will play back slightly faster on NTSC systems than on PAL systems. Faster
CPU clock speeds, however, don't affect the speed of the Paula chip.
