# Metal kernels

Arkestra compiles every `.metal` source named in `pipelines` when the plugin is first used, off the render thread.
It prepends:

```metal
#include <metal_stdlib>
using namespace metal;
```

Including `<metal_stdlib>` again yourself is harmless. Don't `#include` anything else: there's only one source file
per pipeline. The `ark` namespace is reserved.

## Binding

- Arguments are bound **by name** through reflection. The `textures`/`buffers` keys in `ctx.computePass` must equal
  the kernel's argument names; `[[texture(n)]]` and `[[buffer(n)]]` indexes only need to be unique.
- Uniforms always arrive as `constant Uniforms &uniforms [[buffer(0)]]` (any struct name; the argument name must
  be `uniforms`).
- Every texture and buffer argument the kernel declares must be bound, or `computePass` throws and names the
  missing ones. Buffer arguments other than `uniforms` can only take apiVersion 2 host data views.
- Read with `texture2d<float, access::sample>` (or `access::read`), write with `texture2d<float, access::write>`.
  To read and write the same pixel of one texture, use a ping-pong pair instead of `access::read_write`.
- Thread position: `uint2 gid [[thread_position_in_grid]]`. Use `uint3` for 3D grids, `uint` for 1D.

## Uniform layout

Uniforms are packed as consecutive 32-bit floats in the order of the keys of the `uniforms` object:

```js
uniforms: { amount: 1, center: [0.5, 0.5], tint: [1, 1, 1], count: 0 }
```

```metal
struct Uniforms {
    float amount;
    packed_float2 center;   // never float2: it would add padding
    packed_float3 tint;     // never float3: it is 16 bytes in Metal
    float count;            // ints, bools and select indexes arrive as floats
};
```

At most 256 floats per pass. Mismatched order or an unpacked vector type gives garbage values, not an error. Check this first when a parameter
"does nothing" or two parameters seem swapped. Convert in the kernel: `int n = int(uniforms.count);`,
`bool on = uniforms.flag > 0.5;`.

## Coordinates

- `gid` (0,0) is the first texel of the texture. Sampling `ctx.input()` at `(float2(gid) + 0.5) / size` reproduces
  the input, so normalised `uv` from `gid` and texture coordinates are the same space.
- apiVersion 2 vision coordinates are normalised 0…1 in that same space (`gid.y = 0` is their `y = 0`), so
  `point * size` gives pixels directly.
- Text atlases and `atlas.bounds` have row 0 at the top: flip `y` when you map them onto the output.
- For aspect-correct shapes: `float2 p = (float2(gid) - 0.5 * size) / size.y;`.

## Typical kernel skeleton

```metal
struct Uniforms { float amount; };

kernel void myKernel(texture2d<float, access::sample> src [[texture(0)]],
                     texture2d<float, access::write> dst [[texture(1)]],
                     constant Uniforms &uniforms [[buffer(0)]],
                     uint2 gid [[thread_position_in_grid]]) {
    uint w = dst.get_width(), h = dst.get_height();
    if (gid.x >= w || gid.y >= h) return;
    constexpr sampler s(filter::linear, address::clamp_to_edge);
    float2 uv = (float2(gid) + 0.5) / float2(w, h);
    float4 c = src.sample(s, uv);
    dst.write(c, gid);
}
```

## Performance and safety

- Dispatch one thread per output pixel of the texture you write: `ctx.dispatch(pass, w, h)` where `w`/`h` match that
  texture (`ctx.size[0] * 0.5` for a half-scale texture).
- Tie every loop bound to a constant or to a clamped parameter (`max` is enforced). An unbounded GPU loop can hang
  the GPU for the whole app.
- Use `half`-precision formats (`rgba16f`, `r16f`) unless a simulation needs `rgba32f`.
- Compile errors show up as `file:line:column` problems, with lines counted in your file (not the prelude).
- Output alpha matters: Arkestra composites layers, so a generator that should cover the frame writes `a = 1`.
