# pixtreme-transport-nvdec

`pixtreme-transport-nvdec` provides `pixtreme.transport.nvdec`, a streaming H.265 decoder for [pixtreme](https://github.com/sync-dev-org/pixtreme). For each picture (a frame or a field) it delivers the decoded pixels, motion vectors and prediction residual together. The decoder accepts complete Annex B access units, independently of the protocol or file format carrying the stream.

One wheel includes the Python binding, `libpixtreme_nvdec.so`, public C headers, a header-only C++ wrapper and CMake package configuration. Python callers receive owned CuPy arrays; C and C++ callers can borrow decoded GPU surfaces.

## Install

Install from PyPI:

```bash
pip install pixtreme-transport-nvdec
```

For a uv project, use `uv add pixtreme-transport-nvdec`. The installer resolves the `cupy-cuda13x[ctk]==14.2.0` dependency from PyPI. The transport core is not required for the decoder's Python API. The same wheel is also attached to each [GitHub Release](https://github.com/sync-dev-org/pixtreme-transport-nvdec/releases) with its SHA-256 checksum.

## Supported environments

The wheels support Linux x86_64 with glibc 2.28 or later (`manylinux_2_28_x86_64`) and CPython 3.12 or later through the stable Python ABI. Runtime decoding requires an NVIDIA GPU and a driver supporting the CUDA 13 runtime, including in WSL2 and containers. Native GPU code targets sm 86, sm 89 and sm 120. The NVIDIA driver is supplied by the runtime environment.

## C and C++ APIs

The shared library exposes C ABI version 7 through
`pixtreme/nvdec/c/decoder.h`. The header-only C++17 wrapper is
`pixtreme/nvdec/decoder.hpp`. Both interfaces are independent of the transport
core and require no CUDA SDK headers. Add the installed
`pixtreme/transport/nvdec` directory to `CMAKE_PREFIX_PATH`, then use:

```cmake
find_package(pixtreme_transport_nvdec CONFIG REQUIRED)
target_link_libraries(your_target PRIVATE pixtreme::transport_nvdec)
```

C callers check `pixtreme_transport_nvdec_abi_version()` against
`PIXTREME_TRANSPORT_NVDEC_ABI_VERSION` before other library calls. C++ callers
can use `require_compatible_abi()`; `Decoder` also checks during construction.
C functions use opaque handles, records beginning with `struct_size`, and
status/error outputs. Initialize `struct_size` to `sizeof(record)`. An error
handle describes the reason, whether it is fatal, and its message. Release
owned handles with `pixtreme_transport_nvdec_handle_release()`; the public C
headers document borrowed views and the required prefixes of sized records.
The C++ wrapper translates errors into `DecoderError` and manages references.

Rebuild native consumers against the ABI 7 headers and library together.
ABI 7 adds field/layer metadata, output-layer-set selection, surface formats and a mandatory `chroma_bit_depth` field to the
picture descriptor; `bit_depth` describes luma. C++ callers compile the
wrapper into their own binary. C callers use the
`pixtreme_transport_nvdec_` functions directly. Consumers may use their own
C++ compiler and standard library. Existing Python names retain their meanings.

In the C++ wrapper,
`Decoder(device, output_surfaces, threads=0, output_layer_set_index=0)` requires an explicit positive
surface capacity. Hardware decoder creation occurs on the first sequence; an
unsupported capacity fails that `decode` with a fatal `DecoderError`.
`AccessUnit` owns Annex B bytes, optional Annex B or hvcC configuration, and
optional timing. Decimal strings preserve timestamp precision. No packet
type, random-access flag, or discontinuity flag is required.

For same-size two-layer Scalable Main or Scalable Main 4:4:4 SHVC and two-view Multiview Main
MV-HEVC, submit each complete AU with all its layers. The fixed
`output_layer_set_index` selects the VPS output layer set. Its default, zero,
returns base-layer pictures through the single-layer decode path, including
field pictures. Each output exposes read-only `nuh_layer_id()` (Python:
`nuh_layer_id`); ordinary single-layer pictures also have layer ID zero.
Output AU bytes, AU indices and timestamps refer to the complete original
input, including NALs excluded from decoding. In C, the index and layer ID
are appended int64 fields in `decoder_options` and `frame_info`; a shorter
options `struct_size` omits the index and selects zero.

Nonzero indices output the pictures selected by the VPS output flags,
including sets that output both layers or only the upper SHVC layer. Pictures
with the same POC are returned in increasing layer-ID order. Reference-only
pictures are decoded without consuming output loans; the caller's surface
capacity applies to the entire decoder. An index absent from a newly active
VPS reports its exact value and absence as a recoverable error. Later AUs
using a VPS that contains the index can still be decoded.
Scalable Main 4:4:4 accepts matching 8-bit 4:2:0, 4:2:2 or 4:4:4 layers,
subject to the active GPU's NVDEC capability query. Its 4:4:4 pictures use
YUV444 surfaces. Spatially different SHVC layers, unequal layer chroma
formats, non-eight-bit Scalable Main 4:4:4, Scalable Main 10, ten-bit
MV-HEVC, monochrome and more than two layers are rejected at parameter sets.

`decode`, `discontinuity`, and `flush` return `DecodedFrame` values in display
order with CUDA completion events. Wait on each event before reading its GPU
arrays. Each frame's read-only `slice_type()` returns its first slice segment's
HEVC value (0 = B, 1 = P, 2 = I). Read-only `nal_unit_type()` returns the
picture's slice-segment NAL unit type as an integer, including IDR (19, 20),
CRA (21), BLA (16–18), and non-IRAP types. Read-only `vui_timing()` returns an
optional `VuiTiming` with the picture's SPS `num_units_in_tick` and `time_scale`
as unsigned 32-bit integers. It returns `std::nullopt` when VUI is absent or
does not signal timing; no default or frame-period conversion is applied.
`picture` borrows an NVDEC surface with plane layout and
optional VUI colour values. NV12/P016 describe 4:2:0, NV16/P216 describe 4:2:2,
and YUV444/YUV444_16Bit describe three 4:4:4 planes. Two-byte samples store
each plane's d significant bits at bits 16-d through 15, with zero lower bits.
Use `bit_depth` for Y and `chroma_bit_depth` for Cb and Cr; storage determines
the byte count of every sample even when the two depths differ.
Its pointer and byte count span the coded surface. Its width, height, and
plane dimensions describe the SPS conformance window. Read each plane from
`pointer + plane.offset`, using its row and pixel strides; the Y offset can
be nonzero, and chroma starts after the coded luma height. The Python boundary
copies this display region into its owned pictures.
Return the loan
exactly once using `return_picture(lease, read_done_event)` after all GPU reads.
The return operation waits for that event before unmap and remains available
after the `Decoder` object is destroyed. `close()` stops new work immediately;
unreturned loans keep NVDEC alive until their last return. When capacity is
exhausted, input and unmapped pictures are deferred without waiting for a loan.
Return loans, then call `decode` or `flush` to collect deferred pictures. Repeat
`flush` until it returns an empty vector. `raw_mv()` returns the HEVC int32 motion
array; `raw_residual()` returns int16 Y/Cb/Cr arrays, saturating the final
reconstructed residuals to `[-32768, 32767]`. RDPCM and cross-component
prediction complete before saturation; cross-component prediction uses
unsaturated Y. `mv()` returns the processed
float32 motion array `(2, H, W)`, and `residual()` returns the processed float32
residual `(1, H, W)`. `residual()` divides the same saturated Y residuals by the
picture's SPS VUI luma range: `219 * 2^(bit_depth - 8)` for limited or
unsignalled range, and `255 * 2^(bit_depth - 8)` for full range.
`mv()` and `residual()` produce the values of version 4 of the
pixtreme.transport `hevc-mv-residual` profile.
`raw_residual()`, `mv()`, and `residual()` allocate
independent arrays on every call,
with their own completion events. Keep arrays alive until GPU reads finish.
`close()` never waits for a caller to return a loan. When no loans remain, it
finishes its own GPU work and driver resource release before returning.
Host completion waits query CUDA events and sleep for 50 microseconds between
pending queries, including picture returns and resource cleanup.

## Python API

The Python boundary calls the C ABI through the C++ wrapper, copies each borrowed surface into
owned CuPy Y/Cb/Cr planes, returns the surface after the copy, and waits for
completion before returning a tuple of pictures:

```python
from pixtreme.transport.nvdec import DecoderSession

with DecoderSession(threads=0, device=0, output_layer_set_index=0) as decoder:
    pictures = decoder.decode(
        access_unit_bytes,
        configuration=configuration_bytes,  # optional
        configuration_format="annexb",       # or "hvcc"
        pts=None, dts=None, duration=None, time_base=(1, 90000),
    )
    pictures += decoder.flush()
```

`decode` accepts CPU bytes, bytearray, or memoryview. Missing timing stays
missing; `time_base` is an optional integer `(numerator, denominator)` tuple.
Call `discontinuity()` before the first AU after a transport interruption.
Pictures expose metadata, including read-only `slice_type` with the same HEVC
values, read-only integer `nal_unit_type`, and read-only `vui_timing` as an
integer `(num_units_in_tick, time_scale)` tuple or `None` when not signalled,
plus `pictures` and the functions
`raw_mv()`, `raw_residual()`, `mv()`, and `residual()`. The raw functions expose
HEVC values; the processed functions provide the primary output.
The Python API has no surface loan or return operation and does not import the
transport core. `DecoderError` has `reason` and `fatal` attributes.

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
Inter-layer vectors and references remain available in the raw motion output.

## Licenses

The decoder's own code is covered by the MIT license in [LICENSE](LICENSE). The wheel includes the complete license texts for nanobind (BSD-3-Clause), its bundled robin_map (MIT), the ffnvcodec headers (MIT) and the HM tables (BSD-3-Clause) in its `dist-info/licenses/` directory.

The C++ shared library and Python extension statically link the NVIDIA CUDA Runtime. That portion is governed by the [NVIDIA CUDA Toolkit EULA](https://docs.nvidia.com/cuda/eula/index.html) and is outside the permissions granted by this package's licenses. The wheel includes a CUDA runtime notice alongside the license texts.
