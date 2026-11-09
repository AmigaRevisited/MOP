# MOP

A music playback format designed for the Amiga.

See [Specification.md](Specification.md) for technical details on the format.

## Background

The MOP format was born out of attempts to convert Sierra game music for
the Commodore Amiga to something that could be efficiently played by a
re-implemented adventure game engine. The Space Quest IV soundtrack was the
original target, with its orchestral style, although I also wanted it to work
for sequenced music such as in Police Quest III and the general Amiga MOD scene.

Sierra music has some particular features/constraints:

* It was originally stored in MIDI format, with a constant 60 ticks per second
* There may be repetition within a channel, but music often builds with new
  elements as it progresses, so there isn't much repetition across all channels
  at once
* There are often long periods where a channel goes unused, and it's not
  uncommon for channels to go unused altogether
* The "instruments" containing the samples are reused throughout the game,
  though there are some that are specific to just one or two music tracks
* The original player applied attack-decay-sustain-release (ADSR) envelopes on
  the fly, so volume was often changing per tick
* The original player applied arbitrary pitch bending on the fly, and the
  resulting sample period for any given note often doesn't line up with the
  standard table of sample periods used by MOD players
* Sound effects often interrupt playback on a channel, which then resumes
  music playback after the sound
* Perhaps most unusually, secondary "music" tracks can be played that override
  channels from the base track, such as the opening of the time rip in the SQ4
  intro and the simultaneous music from the shops and Skate-O-Rama in the mall
  in SQ4
* A "control" channel includes things like loop points and game cues that
  trigger events in the game

I initially managed to shoehorn some tracks into the MOD format, but over time
it became clear that it wasn't the best fit for this kind of music. Ultimately
I wanted a kind of hybrid between the MIDI and MOD formats:

* Data separated out per channel, so that space isn't wasted for tracks that
  only use one or two channels, and channels can easily be temporarily swapped
  out as a whole
* Playback at 60 ticks per second, where each tick can have "pre-baked" events
  that modify any or all of the sample, sample period, and volume to any
  arbitrary value
* MIDI-style deltas for each event, so that long periods of time where nothing
  changes can be represented by a number of ticks to wait
* A dedicated metadata channel for events that don't apply to playback of
  samples on a particular hardware channel
* Distinction between internally and externally stored samples, so that the
  playback engine in the game could easily reference the shared sample set
  (with the samples loaded in any order, not an order specific to the song)
  as well as samples specific to the individual song
* Support for both 1) storing music data and samples separately,
  and 2) storing them together for distribution of a single song

A nice-to-have would be if the format followed general IFF conventions to be
a good citizen on the Amiga. For example, samples embedded in the file could
be in standard 8SVX format for easy extraction, identification, preservation
of author and copyright notices, etc.

I didn't immediately find anything suitable when I looked around. The closest
I came across was the [Light Speed Player](https://github.com/arnaud-carre/LSPlayer)
(LSP) format, which is exquisitely optimized for speed of playback, and has
a similar notion of pre-baked instructions that are processed with minimal
overhead at a constant frame rate. But LSP is very much tailored to the
demoscene, and to support the features I'd need for the game engine (e.g.
channel replacement, global volume, game cues, individual sample relocation),
I wouldn't be able to use the player or the format as-is, so in the end I wound
up with a distinct format. One concept that I borrowed from LSP is the split
byte/word streams in the channel data.

For Sierra-style game music, the MOP format tends to be significantly smaller
than equivalent MOD song data, and often smaller than the LSP standard format.
I also expect that one could implement a relatively low-overhead player for it
(compared to a typical MOD player), but I'm not a demoscene coder and I haven't
spent a lot of time optimizing playback.
