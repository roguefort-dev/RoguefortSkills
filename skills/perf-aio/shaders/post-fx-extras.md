# Post-FX extras

Fills the gaps in [techniques/post-processing.md](techniques/post-processing.md): production blur/bloom pyramids, more tonemappers, and LUT grading. GLSL. Port to HLSL/WGSL with the rules in the root `SKILL.md`. All inputs are **linear** HDR unless noted.

Most of the code below is written from the published references (cited per section). Before shipping, compare the output against the engine's built-in implementation or the reference image.

## Separable Gaussian with bilinear tap merging

Uses one bilinear fetch to read two texels with the right weights (Rákos, "Efficient Gaussian blur with linear sampling"). A 9-tap Gaussian costs 5 fetches per pass. Run horizontally, then vertically.

```glsl
uniform sampler2D src;
uniform vec2 texel;      // 1.0 / srcSize
uniform vec2 dir;        // (1,0) then (0,1)
const float offs[3] = float[](0.0, 1.3846153846, 3.2307692308);
const float wts[3]  = float[](0.2270270270, 0.3162162162, 0.0702702703);

vec3 blur9(vec2 uv) {
  vec3 c = texture(src, uv).rgb * wts[0];
  for (int i = 1; i < 3; i++) {
    vec2 o = dir * texel * offs[i];
    c += texture(src, uv + o).rgb * wts[i];
    c += texture(src, uv - o).rgb * wts[i];
  }
  return c;
}
```

Wider radius → blur a downsampled copy rather than adding taps.

## Dual-Kawase blur

Marius Bjørge, "Bandwidth-Efficient Rendering" (SIGGRAPH 2015). Each downsample pass halves resolution; each upsample pass doubles it. 2–5 levels give very wide, smooth blurs at a fraction of Gaussian cost. It is ideal for mobile, UI backdrop blur, and bloom.

```glsl
// DOWN: render into a target half the size of src. texel = 1.0 / srcSize.
vec3 kawaseDown(sampler2D src, vec2 uv, vec2 texel, float offset) {
  vec2 h = texel * 0.5 * offset;
  vec3 s = texture(src, uv).rgb * 4.0;
  s += texture(src, uv - h).rgb;
  s += texture(src, uv + h).rgb;
  s += texture(src, uv + vec2(h.x, -h.y)).rgb;
  s += texture(src, uv - vec2(h.x, -h.y)).rgb;
  return s / 8.0;
}

// UP: render into a target twice the size of src. texel = 1.0 / srcSize.
vec3 kawaseUp(sampler2D src, vec2 uv, vec2 texel, float offset) {
  vec2 h = texel * 0.5 * offset;
  vec3 s = texture(src, uv + vec2(-h.x * 2.0, 0.0)).rgb;
  s += texture(src, uv + vec2(-h.x, h.y)).rgb * 2.0;
  s += texture(src, uv + vec2(0.0, h.y * 2.0)).rgb;
  s += texture(src, uv + vec2(h.x, h.y)).rgb * 2.0;
  s += texture(src, uv + vec2(h.x * 2.0, 0.0)).rgb;
  s += texture(src, uv + vec2(h.x, -h.y)).rgb * 2.0;
  s += texture(src, uv + vec2(0.0, -h.y * 2.0)).rgb;
  s += texture(src, uv + vec2(-h.x, -h.y)).rgb * 2.0;
  return s / 12.0;
}
```

`offset` (≈1.0–3.0) widens the blur without extra passes. Radius grows with level count; to animate blur strength smoothly, animate `offset`.

## Mip-chain bloom (CoD: Advanced Warfare style)

Jorge Jimenez, "Next Generation Post Processing in Call of Duty: Advanced Warfare" (SIGGRAPH 2014). This is the modern default and the basis of Unreal's and Unity's high-quality bloom.

1. **Prefilter** (optional): a soft threshold/knee on the HDR scene. Physically based bloom skips the threshold and uses a small mix weight (~0.04) instead.
2. **Downsample** the scene into a mip chain (e.g. 6 levels) with the 13-tap filter. On the **first** downsample only, use a Karis average (weight each 2×2 group by `1 / (1 + luma)`) to kill fireflies.
3. **Upsample** from the smallest mip back up with a 3×3 tent, **additively blending** into the next larger mip.
4. **Composite**: `scene = mix(scene, bloom, strength)` (energy-conserving) or `scene += bloom * strength`, **before** tonemapping.

```glsl
// 13-tap downsample. texel = 1.0 / srcSize, output is half size.
vec3 down13(sampler2D s, vec2 uv, vec2 t) {
  vec3 a = texture(s, uv + t * vec2(-2,  2)).rgb, b = texture(s, uv + t * vec2(0,  2)).rgb, c = texture(s, uv + t * vec2(2,  2)).rgb;
  vec3 d = texture(s, uv + t * vec2(-2,  0)).rgb, e = texture(s, uv).rgb,                   f = texture(s, uv + t * vec2(2,  0)).rgb;
  vec3 g = texture(s, uv + t * vec2(-2, -2)).rgb, h = texture(s, uv + t * vec2(0, -2)).rgb, i = texture(s, uv + t * vec2(2, -2)).rgb;
  vec3 j = texture(s, uv + t * vec2(-1,  1)).rgb, k = texture(s, uv + t * vec2(1,  1)).rgb;
  vec3 l = texture(s, uv + t * vec2(-1, -1)).rgb, m = texture(s, uv + t * vec2(1, -1)).rgb;
  return e * 0.125 + (a + c + g + i) * 0.03125 + (b + d + f + h) * 0.0625 + (j + k + l + m) * 0.125;
}

// 3x3 tent upsample. r = filter radius in UV (e.g. 0.005), output blended additively.
vec3 upTent(sampler2D s, vec2 uv, float r) {
  vec3 a = texture(s, uv + vec2(-r,  r)).rgb, b = texture(s, uv + vec2(0,  r)).rgb, c = texture(s, uv + vec2(r,  r)).rgb;
  vec3 d = texture(s, uv + vec2(-r,  0)).rgb, e = texture(s, uv).rgb,               f = texture(s, uv + vec2(r,  0)).rgb;
  vec3 g = texture(s, uv + vec2(-r, -r)).rgb, h = texture(s, uv + vec2(0, -r)).rgb, i = texture(s, uv + vec2(r, -r)).rgb;
  return (e * 4.0 + (b + d + f + h) * 2.0 + (a + c + g + i)) / 16.0;
}
```

Use `R11G11B10F` mips, and clamp NaN/Inf in the prefilter: one bad pixel otherwise blooms across the screen.

## Tonemappers

Apply exposure first (`c *= exposure`), then the tonemapper. Grade after tonemapping (LUT) or before it in log space, then apply the display OETF (sRGB) unless the render target does that for you.

**Reinhard (extended).** `w` is the white point that maps to 1.0.
```glsl
vec3 reinhardExt(vec3 c, float w) { return c * (1.0 + c / (w * w)) / (1.0 + c); }
```
Applying Reinhard per channel desaturates highlights. Applying it to luminance keeps hue but can clip channels.

**Hable / Uncharted 2 filmic** (John Hable, GDC 2010).
```glsl
vec3 hableCurve(vec3 x) {
  const float A = 0.15, B = 0.50, C = 0.10, D = 0.20, E = 0.02, F = 0.30;
  return ((x * (A * x + C * B) + D * E) / (x * (A * x + B) + D * F)) - E / F;
}
vec3 hable(vec3 c) {
  const float W = 11.2, exposureBias = 2.0;
  return hableCurve(c * exposureBias) / hableCurve(vec3(W));
}
```

**ACES fitted** (Stephen Hill's RRT+ODT fit, BakingLab). This is the formula three.js `ACESFilmicToneMapping` uses. three.js pre-scales the input by `1.0 / 0.6`. The GLSL matrices are column-major.
```glsl
const mat3 ACES_IN  = mat3(0.59719, 0.07600, 0.02840,  0.35458, 0.90834, 0.13383,  0.04823, 0.01566, 0.83777);
const mat3 ACES_OUT = mat3(1.60475, -0.10208, -0.00327, -0.53108, 1.10813, -0.07276, -0.07367, -0.00605, 1.07602);
vec3 rrtOdtFit(vec3 v) {
  vec3 a = v * (v + 0.0245786) - 0.000090537;
  vec3 b = v * (0.983729 * v + 0.4329510) + 0.238081;
  return a / b;
}
vec3 acesHill(vec3 c) { return clamp(ACES_OUT * rrtOdtFit(ACES_IN * c), 0.0, 1.0); }
```
The Narkowicz one-liner `(x*(2.51*x+0.03))/(x*(2.43*x+0.59)+0.14)` is a cheaper, over-saturated luminance fit. ACES shifts hues (bright reds → orange/yellow) and crushes shadows; judge it against Neutral/AgX for your art.

**Khronos PBR Neutral.** It preserves base-color hue and saturation, which makes it the best choice for product/UI-like art. three.js `NeutralToneMapping` uses it.
```glsl
vec3 pbrNeutral(vec3 color) {
  const float startCompression = 0.8 - 0.04;
  const float desaturation = 0.15;
  float x = min(color.r, min(color.g, color.b));
  float offset = x < 0.08 ? x - 6.25 * x * x : 0.04;
  color -= offset;
  float peak = max(color.r, max(color.g, color.b));
  if (peak < startCompression) return color;
  const float d = 1.0 - startCompression;
  float newPeak = 1.0 - d * d / (peak + d - startCompression);
  color *= newPeak / peak;
  float g = 1.0 - 1.0 / (desaturation * (peak - newPeak) + 1.0);
  return mix(color, vec3(newPeak), g);
}
```

**AgX** (Troy Sobotka's AgX; minimal polynomial fit by Benjamin Wrensch). It desaturates gracefully toward white with no hue skews and is Blender 4's default. The input is linear Rec.709. `agx()` returns a display-encoded value; `agxEotf` linearizes it again for pipelines that apply the sRGB OETF afterwards (omit the `pow` if you write the result straight to an 8-bit non-sRGB target). three.js ships this as `AgXToneMapping`.
```glsl
vec3 agxContrast(vec3 x) {
  vec3 x2 = x * x, x4 = x2 * x2;
  return 15.5 * x4 * x2 - 40.14 * x4 * x + 31.96 * x4 - 6.868 * x2 * x + 0.4298 * x2 + 0.1191 * x - 0.00232;
}
vec3 agx(vec3 v) {
  const mat3 m = mat3(0.842479062253094, 0.0423282422610123, 0.0423756549057051,
                      0.0784335999999992, 0.878468636469772, 0.0784336,
                      0.0792237451477643, 0.0791661274605434, 0.879142973793104);
  const float minEv = -12.47393, maxEv = 4.026069;
  v = m * v;
  v = clamp(log2(max(v, vec3(1e-10))), minEv, maxEv);
  v = (v - minEv) / (maxEv - minEv);
  return agxContrast(v);
}
vec3 agxEotf(vec3 v) {
  const mat3 mi = mat3(1.19687900512017, -0.0528968517574562, -0.0529716355144438,
                       -0.0980208811401368, 1.15190312990417, -0.0980434501171241,
                       -0.0990297440797205, -0.0989611768448433, 1.15107367264116);
  v = mi * v;
  return pow(max(v, vec3(0.0)), vec3(2.2));
}
```

Choosing: **Neutral** for faithful albedo and UI-adjacent art, **AgX** for photographic/high-dynamic-range scenes, **ACES** for the "filmic contrast" look, **Hable** for a gentle, controllable curve, and **Reinhard** as a cheap baseline.

## LUT color grading

Grade in a DCC tool by applying adjustments to an identity LUT image, then apply the result as a lookup.

**3D texture LUT** (size N, typically 32). Offset by half a texel so 0 and 1 hit the texel centers:
```glsl
vec3 lut3D(sampler3D lut, vec3 c, float N) {
  vec3 uvw = clamp(c, 0.0, 1.0) * ((N - 1.0) / N) + 0.5 / N;
  return texture(lut, uvw).rgb;
}
```

**2D strip LUT** (N×N slices laid side by side: width N·N, height N; blue selects the slice). Filter manually between the two blue slices:
```glsl
vec3 lutStrip(sampler2D lut, vec3 c, float N) {
  c = clamp(c, 0.0, 1.0);
  float b = c.b * (N - 1.0);
  float s0 = floor(b), s1 = min(s0 + 1.0, N - 1.0), f = b - s0;
  vec2 texel = vec2(1.0 / (N * N), 1.0 / N);
  vec2 rg = c.rg * (N - 1.0) * texel + 0.5 * texel;
  vec3 a = texture(lut, vec2(rg.x + s0 / N, rg.y)).rgb;
  vec3 d = texture(lut, vec2(rg.x + s1 / N, rg.y)).rgb;
  return mix(a, d, f);
}
```
Flip `rg.y` if the LUT image stores green top-down. Apply the LUT to the **tonemapped, display-encoded** color (the space it was authored in), or to log-encoded HDR if the LUT was authored for that.

## Finish: dither

After quantization to 8-bit, gradients band. Add ±0.5 LSB of noise just before output (interleaved gradient noise or blue noise):
```glsl
float ign(vec2 p) { return fract(52.9829189 * fract(dot(p, vec2(0.06711056, 0.00583715)))); }
color += (ign(gl_FragCoord.xy) - 0.5) / 255.0;
```
