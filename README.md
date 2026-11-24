# Wireframe API standard
## Motivation
While developing Nanowarp, I really needed some feedback on algorithms and 
bodged analysis output either through PNG generation or by outputting audio.
[Vamp plugin format](https://www.vamp-plugins.org/) exists, 
together with [Sonic Visualiser](https://www.sonicvisualiser.org/) (sic!) software, 
but for some reason it is a DLL-based format that requires you to link with C++ code, 
which is painful in Go and, really, any other language, and only on Linux it's simply painful and not agonizing. 
This is nonsense, you just need a way to send data, and the simplest and language and platform agnostic way to do it are TCP and UDP.

## Description
Wireframe is a mixed textual/binary serialization and file format. 
Header is written in HTTP-like notation and supports nesting. 
Data consists of time-coded, self-synchronizing sequence of frames. 
Line breaks in text parts are `\n`, field separator is `: `, 
tabulation level is used for nesting and corresponding amount of `\t` specifies the relation to the parent. 
Binary part is encoded in little-endian. Header is separated from data by `\n\n`.

Default host's listen ports are 44444/udp and 44445/tcp.

Time codes in a valid Wireframe stream are monotonic. 
A host **can not** cut excessive spaces, values are always separated using a single space. 
Stream is considered invalid otherwise.

A stream can represent only one channel of audio and only one result. 
Currently I do not know should Wireframe support more.

TBD: host to service/process language, program orchestration, plugin finder, etc.

The general syntax is:
```
github.com/neputevshina/wireframe <version>
<frame type> <sample type> <size of frame OR TBD sizes of lines for tessellation>
<tag>: <value>
<tag>: <value>
<tag>:
	<tag>: <value>
	<tag>: <value>

{<int64(-1)><int64 time code (samples)><#size of frame samples of data>}*
```

Example header:
```
github.com/neputevshina/wireframe v0.1.0
uniform float32 3501
Request ID: 1
Process: PVDR ridge detector
Vendor: neputevshina
Arguments:
    Window: Hann
    Grain: 3500
    Nfft: 7000
    Greedy?: false

```

A Wireframe host **must** support all of the following tag, sample and frame types.

### Tag types
#### `Request ID`
A unique string of a request to a service. 
Host always must send it to the process, echoed back.
#### `Process`
Name of the plugin.
#### `Vendor`
Creator of the plugin.
#### `Unit`
Only for envelopes: numerical unit of the result. 
Host is expected to support only `Hz` and `dB`.
#### `Arguments`
Nested object, echoing back arguments host had given to the plugin.
#### `Estimated runtime`
Estimated runtime of the process in seconds. Formatted as `39.231s`.
#### `Vertical scale`
Only for `uniform` and `tessellation`: type of the labels on the vertical scale. 
Can have two nested objects:
##### `Range`
Range of the values. 
Two numbers, start and end of the range, separated by space.
Mutually exclusive with `Labels`.
##### `Labels`
Only for `custom`: comma-separated list of strings to be used as labels on the scale. 
First label is used for first bin etc.
Mutually exclusive with `Range`.
#### `Source path`
Path to the original audio file fed to the process.
#### `Source sha256`
SHA-256 hash value of the **whole** original audio file fed to the process.
### Sample types
- `float32`
- `float64`
- `bfloat16`
### Frame types
These are **only** valid values of this field.
- `uniform` — frame type of an STFT
- `samples` — horizontal audio-like samples of data
- `envelope` — single value per frame
- `point cloud` (two words, separated by space) — triplets of: value, bin shift, frame shift; directly corresponds to a reassigned spectrogram
- TBD: `tessellation` — Gabor frames, frame type of a sliced constant-Q transform (sliCQ)
- TBD: `partials` — MPE and sinusoidal/Loris components.
### Vertical scales
#### `bins` (default)
Numbered bins. No more, no less.
#### `linear Hz`
Linear bins in hertz.
#### `log Hz`
Log bins in hertz.
#### `mel`
[Mel scale](https://en.wikipedia.org/wiki/Mel_scale).
#### `Bark`
[Bark scale](https://en.wikipedia.org/wiki/Bark_scale).
#### `MIDI`
MIDI note numbers.
#### `chroma`
12-EDO pitch classes (piano note names).
#### `custom`
Take labels from `Labels` subtag.
