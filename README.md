# pixtreme-transport-nvdec

`pixtreme-transport-nvdec` provides `pixtreme.transport.nvdec`, a streaming decoder of H.265, H.264, VP8, VP9, AV1, MPEG-1, MPEG-2, MPEG-4 Part 2, VC-1 and JPEG for [pixtreme](https://github.com/sync-dev-org/pixtreme). For each H.265 picture (a frame or a field) it delivers the decoded pixels, motion vectors and prediction residual together; for each picture of the other codecs it delivers the decoded pixels. The decoder accepts each codec's unit of one picture (a complete Annex B access unit for H.265 and H.264, a frame for VP8 and VP9, a temporal unit for AV1), independently of the protocol or file format carrying the stream.

One wheel includes the Python binding, `libpixtreme_nvdec.so`, public C headers, a header-only C++ wrapper and CMake package configuration. Python, C and C++ callers borrow each decoded picture as a pixtreme `PixelBuffer` without a copy.

## Install

Install from PyPI:

```bash
pip install pixtreme-transport-nvdec
```

For a uv project, use `uv add pixtreme-transport-nvdec`. The installer resolves the `pixtreme>=2.20.0` dependency from PyPI. CuPy is not a dependency of this package; it is installed as a dependency of pixtreme. The transport core is not required for the decoder's Python API. The same wheel is also attached to each [GitHub Release](https://github.com/sync-dev-org/pixtreme-transport-nvdec/releases) with its SHA-256 checksum.

## Supported environments

The wheels support Linux x86_64 with glibc 2.28 or later (`manylinux_2_28_x86_64`) and CPython 3.12 or later through the stable Python ABI. Runtime decoding requires an NVIDIA GPU and a driver supporting the CUDA 13 runtime, including in WSL2 and containers. Native GPU code targets sm 86, sm 89 and sm 120. The NVIDIA driver is supplied by the runtime environment.

## C and C++ APIs

The shared library exposes C ABI version 8 through
`pixtreme/nvdec/c/decoder.h`. The header-only C++17 wrapper is
`pixtreme/nvdec/decoder.hpp`. Both interfaces are independent of the transport
core and require no CUDA SDK headers. They lend each picture as a
[pixtreme](https://github.com/sync-dev-org/pixtreme) `PixelBuffer`, so they
include pixtreme's public headers and link its shared library. Add the
installed `pixtreme/transport/nvdec` directory and pixtreme's `pixtreme/core`
directory to `CMAKE_PREFIX_PATH`, then use:

```cmake
find_package(pixtreme_transport_nvdec CONFIG REQUIRED)
target_link_libraries(your_target PRIVATE pixtreme::transport_nvdec)
```

### Get the native SDK

Each [GitHub Release](https://github.com/sync-dev-org/pixtreme-transport-nvdec/releases)
provides a prebuilt Linux x86_64 SDK. C and C++ applications can use its
installed CMake package without installing Python wheels or compiling either
library. Obtain pixtreme's SDK separately and extract both SDKs into one prefix.
For a release built with pixtreme 2.20.0, run the following commands in a new
directory, replacing `X.Y.Z` with the decoder release version:

```bash
nvdec_version=X.Y.Z
pixtreme_version=2.20.0
nvdec_release="https://github.com/sync-dev-org/pixtreme-transport-nvdec/releases/download/v$nvdec_version"
pixtreme_release="https://github.com/sync-dev-org/pixtreme/releases/download/v$pixtreme_version"
native_prefix="$PWD/native-sdk"
mkdir -p "$native_prefix" nvdec-download pixtreme-download
(
    cd nvdec-download
    curl -fsSLO "$nvdec_release/SHA256SUMS"
    curl -fsSLO "$nvdec_release/pixtreme_transport_nvdec-$nvdec_version-sdk-manylinux_2_28_x86_64.tar.gz"
    sha256sum --check --ignore-missing SHA256SUMS
    tar -xzf "pixtreme_transport_nvdec-$nvdec_version-sdk-manylinux_2_28_x86_64.tar.gz" -C "$native_prefix"
)
(
    cd pixtreme-download
    curl -fsSLO "$pixtreme_release/SHA256SUMS"
    curl -fsSLO "$pixtreme_release/pixtreme-sdk-$pixtreme_version-manylinux_2_28_x86_64.tar.gz"
    sha256sum --check --ignore-missing SHA256SUMS
    tar -xzf "pixtreme-sdk-$pixtreme_version-manylinux_2_28_x86_64.tar.gz" -C "$native_prefix"
)
cmake -S /path/to/your/application -B build -G Ninja \
    -DCMAKE_BUILD_TYPE=Release \
    "-DCMAKE_PREFIX_PATH=$native_prefix/pixtreme/transport/nvdec;$native_prefix/pixtreme/core"
cmake --build build
```

The application uses `find_package(pixtreme_transport_nvdec CONFIG REQUIRED)`
and links `pixtreme::transport_nvdec` as shown above. Preserve the extracted
paths when deploying the SDKs together:
`pixtreme/transport/nvdec/libpixtreme_nvdec.so` and
`pixtreme/core/lib/libpixtreme.so.2`. The decoder retains its wheel filename,
SONAME and relative `$ORIGIN` runtime paths, and each process loads one
pixtreme shared library. Native applications need neither a Python runtime
nor CUDA SDK headers; decoding still needs the GPU and driver described above.

### Use the public interfaces

C callers check `pixtreme_transport_nvdec_abi_version()` against
`PIXTREME_TRANSPORT_NVDEC_ABI_VERSION` before other library calls. C++ callers
can use `require_compatible_abi()`; `Decoder` also checks during construction.
C functions use opaque handles, records beginning with `struct_size`, and
status/error outputs. Initialize `struct_size` to `sizeof(record)`. An error
handle describes the reason, whether it is fatal, and its message. The reason
`UNSUPPORTED` (C++ `ErrorReason::Unsupported`) reports a capability shortfall:
the GPU's NVDEC cannot decode the stream, or no `PixelBuffer` format represents
its pictures. Malformed input, rejected arguments and stream forms the decoder
excludes by design use `INVALID_INPUT`. Release owned handles with
`pixtreme_transport_nvdec_handle_release()`; the public C headers document
borrowed views and the required prefixes of sized records. The C++ wrapper
translates errors into `DecoderError` and manages references.

Rebuild native consumers against the ABI 8 headers and library together.
ABI 8 replaces the decoder's own picture description, loan identifiers and
return operations with pixtreme's `PixelBuffer`, returns `mv()` and
`residual()` as pixtreme NDArrays, removes the H.265 motion and residual
operations (`raw_mv()`, `raw_residual()`) together with the decoder's own
array type, replaces the frame's CUDA completion event
with pixtreme use records, and adds the codec chosen at creation, an optional
owner record per access unit, the capability-shortfall reason, a support
query, and a presence flag for each H.265 metadata field.
C++ callers compile the wrapper into their own binary. C callers use the
`pixtreme_transport_nvdec_` functions directly. Consumers may use their own
C++ compiler and standard library.

In the C++ wrapper,
`Decoder(device, output_surfaces, threads=0, output_layer_set_index=0, codec=Codec::Hevc)`
requires an explicit positive number of pictures lent at once. A decoder
decodes the one codec chosen at creation; this version decodes H.265
(`Codec::Hevc`), H.264 (`Codec::H264`), VP8 (`Codec::Vp8`), VP9 (`Codec::Vp9`),
AV1 (`Codec::Av1`), MPEG-1 (`Codec::Mpeg1`), MPEG-2
(`Codec::Mpeg2`), MPEG-4 Part 2 (`Codec::Mpeg4`), VC-1 of the Simple, Main and
Advanced profiles (`Codec::Vc1`) and JPEG (`Codec::Jpeg`), and an unknown codec
value is invalid input. `output_layer_set_index` applies to H.265 and must be zero
for the other codecs. Hardware decoder creation occurs on
the first sequence; an unsupported number of surfaces fails that `decode` with
a fatal `DecoderError`. `AccessUnit` owns the bytes of one unit (an Annex B
access unit for H.265 and H.264; one frame for VP8 and VP9, a VP9 superframe
included; one temporal unit for AV1, with or without its temporal delimiter;
one picture for MPEG-1 and MPEG-2, one field
picture of a field pair; one VOP for MPEG-4 Part 2; one frame for VC-1, with or
without its frame start code; one image for JPEG), optional configuration
(Annex B start code headers, hvcC for H.265, avcC for H.264, av1C for AV1, or
STRUCT_C and STRUCT_A as `CodecConfigurationFormat::Rcv` for VC-1 Simple and
Main), optional timing and an optional `pixtreme_owner_t` record of the caller.
Bytes that hold more than one unit are invalid input. Decimal strings preserve timestamp precision. No packet type,
random-access flag, or discontinuity flag is required.

The decoder retains an access unit's owner record once when it queues the
input, never for an input it refuses before queueing, and releases it once
when no picture of that input remains in the decoder and every frame made from
it has been released. A `PixelBuffer` taken from such a frame does not delay
the release. An input that yields no frame releases its owner within the
operation that settles this; inputs left after a fatal error release theirs in
`close`. Releases run outside the decoder's locks with `cudaStreamLegacy`, on
the thread that calls `decode`, `discontinuity`, `flush` or `close`, or that
releases the last frame. Frames show the record read-only through `owner()`.

For same-size two-layer Scalable Main or Scalable Main 4:4:4 SHVC and two-view Multiview Main
MV-HEVC, submit each complete AU with all its layers. The fixed
`output_layer_set_index` selects the VPS output layer set. Its default, zero,
returns base-layer pictures through the single-layer decode path, including
field pictures. Each output exposes read-only `nuh_layer_id()` (Python:
`nuh_layer_id`); ordinary single-layer pictures also have layer ID zero.
Output AU bytes, AU indices and timestamps refer to the complete original
input, including NALs excluded from decoding. In C, the index and the codec
are trailing fields of `decoder_options`; a shorter options `struct_size`
omits them and selects zero, which is H.265.

Nonzero indices output the pictures selected by the VPS output flags,
including sets that output both layers or only the upper SHVC layer. Pictures
with the same POC are returned in increasing layer-ID order. Reference-only
pictures are decoded without lending a surface; the number of pictures lent
at once applies to the entire decoder. An index absent from a newly active
VPS reports its exact value and absence as a recoverable error. Later AUs
using a VPS that contains the index can still be decoded.
Scalable Main 4:4:4 accepts matching 8-bit 4:2:0, 4:2:2 or 4:4:4 layers,
subject to the active GPU's NVDEC capability query. Spatially different SHVC
layers, unequal layer chroma formats, non-eight-bit Scalable Main 4:4:4,
Scalable Main 10, ten-bit MV-HEVC, monochrome and more than two layers are
rejected at parameter sets.

`decode`, `discontinuity`, and `flush` return `DecodedFrame` values in display
order without waiting for GPU completion. A frame carries no CUDA event: the
decoder records its writes on its streams as pixtreme use records of the
picture's `PixelBuffer` and of each NDArray that `mv()` and `residual()`
return, and a reader orders its stream after them. Metadata defined by H.265 syntax is
optional: on H.265 frames only `vui_timing()` and `pic_struct()` can be empty,
and every other field holds a value; frames of the other codecs hold none of
them, and their `mv()` and `residual()` throw a recoverable `InvalidState`
error. An H.264 or MPEG-2 stream whose fields are separate pictures passes one
field's unit per call; the frame of two fields carries its first field's input
values. As libvpx and libaom output, a VP9 input returns its last frame when
that frame is shown, and an AV1 temporal unit the last frame it shows; a hidden
frame returns none, and a frame shown again with `show_existing_frame` carries
the input values of the call that shows it. Each frame's read-only `slice_type()`
returns its first slice segment's HEVC value (0 = B, 1 = P, 2 = I). Read-only
`nal_unit_type()` returns the picture's slice-segment NAL unit type as an
integer, including IDR (19, 20), CRA (21), BLA (16–18), and non-IRAP types.
Read-only `vui_timing()` returns the picture's SPS `num_units_in_tick` and
`time_scale` as unsigned 32-bit integers, or no value when VUI is absent or
does not signal timing; no default or frame-period conversion is applied.
`field_seq_flag()` is zero for an SPS without VUI, as H.265 specifies.

Each frame holds one reference to its picture. `take_picture()` (C:
`pixtreme_transport_nvdec_frame_take_picture`) moves that reference to the
caller once, as a `pixtreme::pixel_buffer` that lends the NVDEC surface
without a copy; a second take is a recoverable `InvalidState` error. The
`PixelBuffer` declares its pixtreme format, bit depth and alignment, the
offset and pitch of each plane, and the display size. 4:2:0 pictures use
`nv12`, `p010` and `p012`; 4:2:2 pictures use `nv16`, `p210` and `p212`; 4:4:4
pictures use `yuv444p`, `yuv444p10msb` and `yuv444p12msb`. Two-byte samples
store d significant bits at bits 16-d through 15, with zero lower bits. The
surface holds the whole coded picture, and each plane's offset points at the
display region's top left. Its use records order reads after the decoder's
writes. Its colour declaration converts the picture's SPS VUI
`colour_primaries`, `transfer_characteristics`, `matrix_coeffs` and
`video_full_range_flag` to pixtreme tokens with pixtreme's H.273 conversion
(`pixtreme_color_h273_to_token`). A value the VUI does not signal, or a code
without a pixtreme token, stays unknown (null, `None` in Python) rather than a
default. A 4:2:0 picture's siting comes from `chroma_sample_loc_type_top_field`,
or from `chroma_sample_loc_type_bottom_field` for a field picture whose
`pic_struct` names the bottom field (2, 10 or 12); it is unknown when the VUI
signals no chroma location, and for 4:2:2 and 4:4:4 pictures.
Streams whose luma and chroma bit depths differ are refused as a
capability shortfall, because a `PixelBuffer` has one bit depth.
Pictures of the other codecs declare the colour values their syntax signals,
or those their standard specifies when it signals none (the C header lists
them per codec); gray JPEG images are lent as `gray8`. JPEG 4:2:2 and 4:4:4
images, which NVDEC outputs only subsampled to 4:2:0, and JPEG images other
than sequential Huffman (baseline and extended) ones, such as progressive images,
which NVDEC does not decode, are refused as a capability shortfall.
After a VP8, VP9 or AV1 frame is refused as a capability shortfall, subsequent inputs are
refused as `unsupported` until a key frame initializes all saved decoding
state or `discontinuity()` discards the references. A non-key frame that needs
the NVDEC decoder to be recreated, and a VP9 non-key frame whose width or
height exceeds the last accepted key frame's width or height, are refused this
way, even if the decoder reserved more space; a discontinuity also forgets the
last accepted key frame's size. Intra-only frames do not resume decoding after
this refusal. A frame that shows again a picture of another size than its
owning decoder's current output is refused for that input alone; this does not
start a wait for a key frame. After a discontinuity, frames that refer to
discarded references are refused as invalid input.

There is no return operation. Release the `PixelBuffer`'s last reference on
the stream of its last read (`pixel_buffer::release(stream)` or
`pixtreme_pixel_buffer_release`); the decoder unmaps the surface once that
stream's preceding work completes, and only then reuses it. The destructor and
`pixtreme_pixel_buffer_destroy` release on the legacy default stream, so after
a last read on another stream, confirm on the host that the read finished
first. Releasing every reference to a frame whose picture was not taken returns
its surface as well. A taken `PixelBuffer` stays valid after its frame and the
decoder are released or closed. When every surface is lent, `decode` succeeds
and defers output without waiting for a return; after releasing a picture, call
`decode` or `flush` to collect deferred pictures. A VP9 or AV1 frame that shows
again a picture whose earlier `PixelBuffer` is still held is deferred the same
way, since NVDEC maps a decoded picture once at a time. Repeat `flush` until it
returns an empty vector.

`is_supported(device, codec, chroma_format, bit_depth, width, height)` (C:
`pixtreme_transport_nvdec_is_supported`) answers whether the device's NVDEC
can decode that codec, chroma format, luma bit depth and coded size and the
decoder can lend such a picture, applying the same conditions as the decode
errors with the reason `Unsupported`. Conditions decided while parsing, such
as the decode surface count, output layer sets and stream forms excluded by
design, are left to `decode`.

`mv()` (C: `pixtreme_transport_nvdec_frame_mv`) returns the processed
float32 motion array `(2, H, W)`, and `residual()` (C:
`pixtreme_transport_nvdec_frame_residual`) returns the processed float32
residual `(1, H, W)`, each as a new `pixtreme::ndarray` (C:
`pixtreme_ndarray_t`) in memory allocated for that call alone, so writing it
never changes a later result. `residual()` divides the picture's signed luma
residual before prediction, saturated to `[-32768, 32767]` after RDPCM and
cross-component prediction, by the picture's SPS VUI luma range:
`219 * 2^(bit_depth - 8)` for limited or unsignalled range, and
`255 * 2^(bit_depth - 8)` for full range. `mv()` and `residual()` produce the
values of version 4 of the pixtreme.transport `hevc-mv-residual` profile. The
call does not wait for the decoder's write, which the NDArray's use records
hold. Before reading on another CUDA stream, order that stream after the write
(`wait`), record the reads (`record_stream`), and release the last reference on
the stream of the last read (`release(stream)` or `pixtreme_ndarray_release`):
the decoder frees the memory after that stream's preceding work and never
waits for it on the host. The release orders the release stream after the
decoder's write, so work queued later on that stream waits for the write even
when the array was released without a read. The destructor and `pixtreme_ndarray_destroy`
release on the legacy default stream, so after a last read on another stream,
confirm on the host that the read finished first. The NDArray stays valid
after its frame and the decoder are released or closed.

`mv()` and `residual()` are the decoder's only motion and residual outputs.
The H.265 motion and the int16 Y, Cb and Cr residuals they are computed from
belong to no public interface; the tests read them through an uninstalled
verification binding to compare the decoder with HM, SHM and HTM. Both
operations stay available while the frame is valid, whether or not its
surface has been returned.

`close()` does not wait for lent pictures: it stops decoding and lending and
frees the resources that are not lent. The surface of a `PixelBuffer` still
referenced is neither reused nor unmapped until its last release; the decoder
then unmaps it after the release stream's work and frees the rest, NVDEC
included. Destroying the NVDEC decoder can block the host until GPU work
queued in the same CUDA context finishes. Host completion waits query CUDA
events or streams and sleep for 50 microseconds between pending queries.

## Python API

The Python boundary calls the C ABI through the C++ wrapper and waits for the
writes in each picture's use records before returning a tuple of
`PictureRecord` objects in display order:

```python
from pixtreme.transport.nvdec import DecoderSession

with DecoderSession(threads=0, device=0, output_layer_set_index=0,
                    output_surfaces=4, codec="hevc") as decoder:
    for record in decoder.decode(
        access_unit_bytes,
        configuration=configuration_bytes,  # optional
        configuration_format="annexb",       # "hvcc", "avcc", "av1c" or "rcv"
        pts=None, dts=None, duration=None, time_base=(1, 90000),
    ):
        picture = record.take_picture()  # a pixtreme.PixelBuffer
```

`output_surfaces` is required and names the number of pictures the decoder can
lend at once. `codec` defaults to `"hevc"`; the other names are `"h264"`,
`"vp8"`, `"vp9"`, `"av1"`, `"mpeg1"`, `"mpeg2"`, `"mpeg4"` (MPEG-4 Part 2),
`"vc1"` and `"jpeg"`, and another
name raises `DecoderError` with reason `"invalid_input"`.
`decode` accepts CPU bytes, bytearray, or memoryview. Missing timing stays
missing; `time_base` is an optional integer `(numerator, denominator)` tuple.
Call `discontinuity()` before the first AU after a transport interruption.

`take_picture()` moves the record's picture to the caller once, as a
`pixtreme.PixelBuffer` that lends the NVDEC surface without a copy; a second
call raises `DecoderError` with reason `"invalid_state"`. Reading attributes,
including `repr`, leaves the picture in the record. The surface returns when
the PixelBuffer's last reference is released, or when a record whose picture
was not taken is released. While `output_surfaces` pictures are lent, `decode`
and `flush` hold later pictures back; release one and call `decode` or `flush`
again, until `flush` returns an empty tuple. A PixelBuffer stays valid after
`close()` and after the session is released.

Records expose metadata, including read-only `slice_type` with the same HEVC
values, read-only integer `nal_unit_type`, and read-only `vui_timing` as an
integer `(num_units_in_tick, time_scale)` tuple or `None` when not signalled,
plus the functions `mv()` and `residual()`.
Fields defined by H.265 syntax are typed as optional; H.265 records show a
value in each of them, except `vui_timing` and `pic_struct` when not signalled.
`mv()` and `residual()` return the wrapper's NDArrays as float32
`pixtreme.NDArray` values after the writes in their use records complete, and
are the only motion and residual outputs. The Python API does not import the
transport core and does not take an owner record.

`is_supported(codec, chroma_format, bit_depth, width, height, device=0)`
answers whether the device's NVDEC can decode such a stream and the decoder
can lend its pictures; `chroma_format` is `"4:0:0"`, `"4:2:0"`, `"4:2:2"` or
`"4:4:4"`, and `width` and `height` are the coded size. `DecoderError` has
`reason` (`"invalid_input"`, `"invalid_state"`, `"unsupported"`,
`"concurrent_operation"` or `"driver_failure"`) and `fatal` attributes;
`"unsupported"` reports the capability shortfall that `is_supported` answers.

For a stream with SPS `field_seq_flag=1`, each output contains one field's
rows at that field picture's display dimensions. Output remains in display
order and does not wait for the complementary field. The decoder does not
weave fields or insert rows. `field_seq_flag` is a read-only bool and
`pic_struct` comes from base picture timing SEI, or the upper layer's
frame-field information SEI. Picture timing directly inside scalable nesting
in that layer's SEI NAL is also supported. The value is `None` when not signalled;
parity is never inferred from POC. C uses `field_seq_flag`, `has_pic_struct`
and `pic_struct` in `frame_info`; C++ provides `field_seq_flag()` and
`pic_struct()` returning bool and `std::optional<int>`. VUI timing values
remain the signalled values without conversion to a field or frame rate.

Processed motion (`mv()`) uses the field's display dimensions and field rows
as its vertical unit. Each usable reference contributes its quarter-pixel
vector divided by 4 and by the POC distance. Opposite field parity does not
add a half-row correction. Two usable lists are averaged; one contributes
alone, and no usable list produces zero (SP-HEVCDEC-105). Only same-layer
temporal references with a known full POC and a nonzero POC distance contribute.

## Licenses

The decoder's own code is covered by the MIT license in [LICENSE](LICENSE). The wheel includes the complete license texts for nanobind (BSD-3-Clause), its bundled robin_map (MIT), the ffnvcodec headers (MIT) and the HM tables (BSD-3-Clause) in its `dist-info/licenses/` directory.

The C++ shared library and Python extension statically link the NVIDIA CUDA Runtime. That portion is governed by the [NVIDIA CUDA Toolkit EULA](https://docs.nvidia.com/cuda/eula/index.html) and is outside the permissions granted by this package's licenses. The wheel includes a CUDA runtime notice alongside the license texts.
