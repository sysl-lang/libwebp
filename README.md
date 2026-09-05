# libwebp

**WebP for sysl — the format that made "PNG or JPEG?" stop being the question.** Its lossy mode is
VP8's intra coder and comes out roughly a quarter smaller than a JPEG of the same quality; its
lossless mode beats PNG by about a fifth and, unlike PNG, it carries an alpha channel in the *lossy*
mode too. One library, two coders, and one dial between them.

```
dependencies {
  libwebp { git = "github.com/sysl-lang/libwebp", version = "0.1.0" }
}
```

`sh.sysl.stb` is the package for PNG, JPEG, BMP, TGA and HDR, and `sh.sysl.png` the one for PNG on
its own. This is the one for anything a browser is going to fetch.

## The four calls

```sysl
import sh.sysl.libwebp.*

val info = probe(bytes)?                       // width, height, alpha — from the header alone
val img  = decode_rgba(bytes)?                 // &Image; img.pixels() is width * height * 4 bytes
val out  = encode_rgba(px, 640, 480, 80.0)?    // &Encoded; out.bytes() is a whole .webp file
val same = encode_lossless_rgba(px, 640, 480)? // the same, with nothing thrown away
```

**`probe` reads the header and nothing else**, so it is the call to make before deciding whether to
decode a file at all — a hostile 16383 × 16383 image is a gigabyte of RGBA, and finding that out
should not cost a gigabyte.

## The API

| | |
|---|---|
| `probe(data)` | an `Info`: `width`, `height`, `has_alpha`, `animated`, `format` |
| `is_webp(data)` | the twelve-byte `RIFF`…`WEBP` header, and nothing else |
| `decode_rgba(data)` / `decode_rgb(data)` | a `&Image` holding libwebp's pixels |
| `decode_rgba_into(data, dst, stride)` / `decode_rgb_into(…)` | into a slice you own; **allocates nothing** |
| `encode_rgba(px, w, h, quality, stride)` / `encode_rgb(…)` | a `&Encoded` holding a whole `.webp` |
| `encode_lossless_rgba(px, w, h, stride)` / `encode_lossless_rgb(…)` | the same, with no quality to pass |
| `Image.pixels()` `.width` `.height` `.channels` `.len()` | |
| `Encoded.bytes()` `.len()` | |
| `Info.rgba_len()` / `.rgb_len()` | how large a slice `decode_*_into` wants |
| `version()` `decoder_version()` `encoder_version()` | `"1.6.0"`, `0x010600` |
| `max_dimension()` | `16383`, which is the format's rather than this build's |

## Who owns the pixels

**libwebp allocates, and `Image` and `Encoded` are the handles that release it.** Both one-shot
families answer storage the library `malloc`'d, and there is no allocator hook in libwebp's public
API to point at storage a program already owns — the `WEBP_USE_*` knobs are build-time macros of the
library rather than run-time parameters, so a binding setting them would be pinned to a build of
libwebp nobody has installed. So the handles hold the library's pointer, `pixels()` and `bytes()` are
views into it, and a `Drop` calls `WebPFree` when the handle goes. It is the shape `sh.sysl.stb` uses
for exactly the same reason, and it costs no copy.

**A caller who wants none of that has `decode_rgba_into`**, which decodes straight into a slice the
program already owns and allocates nothing. It is the call for a loop over many frames, and the one
that would work over an arena.

## Errors say which thing happened

libwebp answers **one null pointer** to a truncated file, a corrupt one, an animation and an
allocation failure alike, and **one zero** to every reason an encode did not happen. Pulling those
apart is most of what this package is.

| | |
|---|---|
| `NotWebP` | libwebp saw enough bytes to be sure these are not one |
| `Truncated` | the input stops before the image does. **A partial download, not a corrupt one** — the difference matters to whatever is retrying |
| `Corrupt` | it is a WebP and the image data is damaged |
| `Animated` | an animated WebP. See below |
| `Unsupported` | a feature this build of libwebp will not decode |
| `Dimensions(w, h)` | outside `1..16383`, which is what fourteen bits of canvas will hold |
| `Shape(needed)` | the pixel slice and the dimensions disagree; `needed` is the figure that works |
| `Quality(q)` | outside `0..100` |
| `NoMemory` | libwebp refused to allocate |
| `Refused` | the encoder said no and says nothing more. The shape and the quality are already checked here, so what is left is the library's own |

**`Truncated` is half this package's own work.** libwebp reports "not enough data" only while the
*header* is incomplete; a file whose header survives and whose image data is cut off decodes to the
same bare null a corrupt one does. So `decode` compares the RIFF header's declared payload against
the slice it was handed, and a file short of its own declaration is `Truncated` before libwebp is
asked. The suite checks four different prefixes of a real file.

**An empty slice is `Truncated` here and `INVALID_PARAM` in libwebp**, which would arrive as
`NotWebP` — a verdict about bytes nobody has seen. A caller reading a socket has an empty buffer
before it has anything else.

## Animation is refused, and refused deliberately

**An animated WebP is a container this package does not read.** Its frames live in `ANMF` chunks that
only `libwebpdemux` — a second library, a second header and a second `pkg-config` name — knows how to
walk, and `WebPDecodeRGBA` on one answers null with no explanation at all.

So **`probe` reports it** and every decode answers `Error.Animated`, whose sentence names what is
missing:

```
this is an animated WebP; reading one needs libwebpdemux, which this package does not bind
```

That ordering is deliberate: a caller can tell an animation from a failure *before* asking for
pixels. Adding real support means binding `demux.h`'s `WebPAnimDecoder`, which is a stateful handle
with its own ownership story; it has not been asked for.

## Lossless is not `exact`, and the difference is worth knowing

libwebp's lossless encoders default to `exact = 0`, which **rewrites the colour channels of fully
transparent pixels** to whatever compresses best — nothing can see them. That is lossless about the
*image* and not about the *bytes*: a round trip returns every alpha value exactly, returns every
visible pixel exactly, and may return different RGB under `alpha == 0`. Reaching the flag needs
`WebPEncode` and a `WebPConfig`, which this package does not bind. The suite pins the behaviour in
both directions rather than hiding it.

## Requirements

libwebp has to be installed, and so do its headers. **Nothing is vendored** — libwebp is a quarter of
a megabyte of C with its own CMake build and hand-written SIMD for four architectures, and every
platform ships it as one package.

```
brew install webp
sudo apt install libwebp-dev
sudo pacman -S libwebp
```

**`webp` is the Homebrew formula name**, so a consumer's own formula wants `depends_on "webp"`. The
build needs no flags — `pkg-config` is asked where the library is.

**`requires { heap = true }`**, and there is no way around it: every decode and every encode is one
allocation the library makes and this package releases. **No `posix`** — nothing here opens a file,
reads a clock or touches a socket. (The *suite* reaches `sysl.process` and `sysl.fs` to drive `cwebp`
and `dwebp`; a `@tests` file's needs are not the package's, and a consumer pays nothing for them.)

**This is not a package for a freestanding target.** It resolves a shared library through
`pkg-config`, which is a claim that the platform has one — see `~/dev/sysl-lang/CLAUDE.md` on what a
capability clause does and does not say.

## Checking the handles are released

A decoded 640 × 480 image is 1.2 MB and nothing in a suite notices one being leaked — the pixels are
still right. What notices is the resident set over a loop:

```sysl
for _ in 0..<50000
    val file = encode_lossless_rgba(src, 32, 32)?
    val back = decode_rgba(file.bytes())?
```

```
/usr/bin/time -l ./probe
```

| iterations | maximum resident set |
|---|---|
| 1,000 | 3.83 MB |
| 5,000 | 5.90 MB |
| 10,000 | 4.85 MB |
| 50,000 | 5.93 MB |
| 200,000 | 5.85 MB |

Flat, and not even monotonic — which is what a program that releases everything looks like. **With
the constructors returning plain values instead of `&Image` and `&Encoded`, the same program at
50,000 reaches 622 MB.** That number is the proof the measurement can fail; without it a flat line
says nothing.

That is why every constructor hands back a box: a destructor runs when a box's strong count reaches
zero and for nothing else, so a `Drop` on a type nobody boxes is never called. **sysl 0.0.105 warns
about it** — `hands back 'Encoded' by value, and 'Encoded' has a 'Drop'` — which is how the unboxed
measurement above was produced deliberately rather than discovered by accident.

## Testing

```
sysl test .
```

**26 tests, no network.** Four of them are the raw layer asserting that sysl and clang agree about
libwebp's memory — every field of `WebPBitstreamFeatures` against the header's own `offsetof`, its
`sizeof`, and the whole `VP8StatusCode` enumeration — so an upstream that reorders one fails a test
rather than reporting somebody else's width for ever. One of those checks that the **ABI version**
guard can be satisfied *and* that it refuses a wrong number, which is what makes the first half mean
anything.

**Two of them drive `cwebp` and `dwebp`**, which is the corroboration that could actually fail: a
lossless file this package writes is decoded by `dwebp` to a PPM compared byte for byte against the
source, a lossless file `cwebp` writes is decoded by this package and compared the same way, and the
two commands' quality-10 and quality-90 output is compared in size against ours at the same
qualities. They are **skipped rather than failed** where the commands are not installed.

**The honest limit on that is worth stating.** `cwebp` and `dwebp` link the same `libwebp` this
package calls, so agreement says nothing about the *format* — both would agree about a mistake in the
library. What it checks is the **binding**: that the width, the height, the stride and the quality
reach the encoder the caller asked for, that pixels come back in the order the header promises, and
that a whole file is what this hands over rather than a fragment. There is no from-scratch WebP
encoder on this machine to be a second oracle: Pillow's plugin and Chrome's decoder are both this
same libwebp.

**Each test was checked to be capable of failing**, by breaking the implementation on purpose and
watching which went red. Every mutation reddened something:

| what was broken | what went red |
|---|---|
| the quality never passed to `encode_rgba` | the quality test |
| the quality never passed to `encode_rgb` | the `cwebp` quality cross-check |
| the row stride ignored | the stride test and the strided-decode test |
| the animation check dropped | the animation test |
| the RIFF completeness check dropped | both truncation tests |
| `decode_rgb` decoding four channels | four tests, including the `dwebp` cross-check |
| the pixel-slice check dropped | the shape test and the strided-decode test |
| the empty-input case folded back into libwebp's | the garbage test |
| the constructors returning unboxed values | 5.9 MB became 622 MB |

Two fixtures had to be replaced during that pass, and both for the reason
`~/dev/sysl-lang/CLAUDE.md` records against zstd — **a fixture that cannot tell the settings apart**:

- **A smooth gradient compresses to nothing at every quality libwebp offers**, so the "a higher
  quality costs more bytes" test would have passed on a binding that never passed the quality. The
  fixture is now half structure and half xorshift noise.
- **A 200 × 150 gradient encodes losslessly to eighty-odd bytes**, so the "probe reads only the
  header" test was handing `probe` most of the file rather than a prefix of it.

And one tolerance was wrong rather than the code: `gradient`'s channels are `(x * 4) % 256`, so every
sixty-four pixels there is a hard step from 252 to 0 — and a lossy coder rings at a step like that,
moving a channel by **121** at quality 95 while being entirely correct. The lossy tolerance test now
uses an image a photograph could be.

**AddressSanitizer** (`SYSL_EXTRA_CFLAGS="-fsanitize=address -g" sysl test .`) is clean, on the suite
and on a program doing fifty thousand round trips, with the instrumentation confirmed rather than
assumed — `nm -u <binary> | grep asan` reports 17 symbols including `__asan_memcpy` and
`__asan_report_load8`. **What it covers is the sysl half only**: this package resolves libwebp
through `pkg_config`, so the objects were compiled by the platform months ago and no flag of ours
instruments them.

## What is deliberately not bound

**The demuxer** (`demux.h`) — `WebPAnimDecoder` and the chunk iterator, which is what reading an
animation, an ICC profile, EXIF or XMP takes. A second library and a second `pkg-config` name.

**The mux** (`mux.h`) — assembling those containers. Same argument.

**The incremental decoder** — `WebPIDecoder`, `WebPIAppend`, `WebPIDecGetRGB`: the shape for showing
an image as it arrives over a socket, one packet at a time. Worth having, and it is a stateful handle
with a different ownership story from anything here.

**`WebPEncode` and `WebPConfig`** — the fifty-odd knobs behind the one-shot encoders: `method`,
`segments`, `filter_strength`, `sns_strength`, `near_lossless`, `alpha_quality`, the presets, and
`exact`. One of those is a real gap rather than a knob and the section above says so.

**`WebPPicture`** and everything hanging off it — cropping, rescaling, the YUV conversions, the
sharp and smart ARGB-to-YUVA paths, and `WebPPictureDistortion`'s PSNR and SSIM. `sh.sysl.stb`'s
resizer covers the scaling case.

**The YUV decoders** (`WebPDecodeYUV`, `WebPDecodeYUVInto`) — three planes with three strides, which
is what a video pipeline wants and nothing else does.

**BGR, BGRA and ARGB orderings.** libwebp offers each of them and they are one `extern` apiece; RGBA
and RGB are what every consumer in this org asks for. Trivial to add when something wants one.

None of these has been asked for.
