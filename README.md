# pixtreme-transport-nvdec

`pixtreme-transport-nvdec` provides `pixtreme.transport.nvdec`, a streaming H.265 decoder for [pixtreme](https://github.com/sync-dev-org/pixtreme). For each frame it delivers the decoded picture, motion vectors and prediction residual together. The decoder accepts complete Annex B access units, independently of the protocol or file format carrying the stream.

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

The shared library exposes C ABI version 6 through
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

To move from ABI 5, rebuild native consumers against the ABI 6 headers and
library together. Existing C++ callers keep the namespace and type names;
they now compile the wrapper into their own binary. C callers use the
`pixtreme_transport_nvdec_` functions directly. Consumers may use their own
C++ compiler and standard library. Python names and behavior are unchanged.

In the C++ wrapper,
`Decoder(device, output_surfaces, threads=0)` requires an explicit positive
surface capacity. Hardware decoder creation occurs on the first sequence; an
unsupported capacity fails that `decode` with a fatal `DecoderError`.
`AccessUnit` owns Annex B bytes, optional Annex B or hvcC configuration, and
optional timing. Decimal strings preserve timestamp precision. No packet
type, random-access flag, or discontinuity flag is required.

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
optional VUI colour values; P016 samples contain ten bits starting at bit 6.
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
array; `raw_residual()` returns int16 Y/Cb/Cr arrays. `mv()` returns the processed
float32 motion array `(2, H, W)`, and `residual()` returns the processed float32
residual `(1, H, W)`. `residual()` divides signed luma residuals by the
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

with DecoderSession(threads=0, device=0) as decoder:
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

## Licenses

The decoder's own code is covered by the MIT license in [LICENSE](LICENSE). The wheel includes the complete license texts for nanobind (BSD-3-Clause), its bundled robin_map (MIT), the ffnvcodec headers (MIT) and the HM tables (BSD-3-Clause) in its `dist-info/licenses/` directory.

The C++ shared library and Python extension statically link the NVIDIA CUDA Runtime. That portion is governed by the [NVIDIA CUDA Toolkit EULA](https://docs.nvidia.com/cuda/eula/index.html) and is outside the permissions granted by this package's licenses. The wheel includes a CUDA runtime notice alongside the license texts.
