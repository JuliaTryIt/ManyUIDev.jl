# Upstream bugs

Defects that live outside this repository and that we work around here.
Each entry records how it was verified, so a future release can be
re-tested against the same evidence rather than guessed at.

---

## 1. `libcimgui` ships without FreeType, so colour emoji cannot render

**Where:** `CImGuiPack_jll` 0.12.2 (the binary behind `CImGui` 8.0.0)
**Affects:** every `cimgui` and `cimguitui` demo mode
**Status:** open upstream, but **lifted on this machine by a local
override**. `~/.julia/packages/CImGuiPack_jll/*/override/lib/libcimgui.dylib`
is a FreeType + HarfBuzz-shaping build, so `_has_freetype()` and
`CImGui.HasShaping()` both return true there and the shaped paths are
live: colour emoji rasterise, and `RenderShapedText` forms the ZWJ and
regional-indicator ligatures. Verified by framebuffer dump, not by eye.

Everything below still describes the STOCK artifact, which is what
anyone without that override gets — and the code keeps both guards, so
it degrades to unshaped text there instead of throwing.

The shipped `libcimgui.dylib` is a plain **stb_truetype** build. Its
rasteriser cannot read COLR layers, sbix strikes or CBDT bitmaps, which
is where all the art in a colour emoji font lives. Emoji therefore draw
as tofu boxes regardless of which font is installed.

Verified on the macOS arm64 artifact:

```console
$ otool -L .../lib/libcimgui.dylib | grep -i freetype     # (nothing)
$ nm -u .../lib/libcimgui.dylib | grep -c _FT_            # 0
$ strings .../lib/libcimgui.dylib | grep -i freetype
FreeType
Requires #define IMGUI_ENABLE_FREETYPE + imgui_freetype.cpp.
```

That last line is Dear ImGui's own error string for the case, compiled
into the binary. The font files are not the problem: `TwemojiCOLR.ttf`
(COLR v0) is present on the dev machine, loads, merges, and still bakes
nothing.

Two consequences, both upstream:

* No colour emoji at all.
* No GSUB shaping, so a ZWJ sequence such as 👨‍👩‍👧‍👦 cannot form its
  single ligature glyph. `ImGuiFreeType::RenderShapedText` is the API
  for that and it is absent here.

**What we do about it.** `_has_freetype()` in
`ManyUICImGui/ext/ManyUICImGuiCImGuiExt.jl` detects the build by symbol
and warns once, so the tofu is explained rather than mysterious. It also
now guards the shaping call: the previous guard was
`isdefined(CImGui, :HasShaping)`, which asks about the *Julia* binding —
always defined by CImGui.jl's generated wrapper — so `HasShaping()` was
reached and threw "could not load symbol" inside the render loop.

Everything that is an ordinary outline glyph is unaffected and does
render: Latin, box drawing, Braille (the `Spinner` frames) and CJK.

**Re-test with:** a `CImGuiPack_jll` built with `IMGUI_ENABLE_FREETYPE`
(and `IMGUI_ENABLE_HARFBUZZ_SHAPING` for the ligatures). Delete the
warning and the guard once `_has_freetype()` returns true.

---

## 2. `HarfBuzz.jl`'s `HarfBuzz_jll` bound made GLFW 3.4 unresolvable

**Where:** `HarfBuzz.jl` (the s-celles fork), `[compat] HarfBuzz_jll = "100.14002"`
**Affects:** the whole CImGui backend — it segfaulted on startup
**Status:** FIXED upstream, in the fork itself — "Support the HarfBuzz 8.x
series, not only 100.x" (#7). The bound is now
`HarfBuzz_jll = "8.5.1, 100.14002"` and `hb_font_is_synthetic` is
guarded by `Libdl.dlsym`. `ManyUIDemos/CImGuiEnv` resolves to GLFW_jll
3.4.1 / HarfBuzz_jll 8.5.1 / Pango_jll 1.58.0, the extension wakes
(`ManyUICImGui.native_available() == true`) and the window opens.

Kept here because a stale `Manifest.toml` reintroduces the whole
failure: it pins the pre-fix commit of the fork, and the resolver then
walks straight back to GLFW_jll 3.3.9. That is exactly what made the
CImGui modes unreachable long after the fix landed. **A `signal 11` in
a CImGui mode is a GLFW_jll version question first** — check it with
`julia --project=CImGuiEnv -e 'using Pkg; Pkg.status("GLFW_jll"; mode=PKGMODE_MANIFEST)'`
and run `just instantiate-cimgui` after `Pkg.update()` if it reads
3.3.9.

The original analysis follows.

`libcimgui` calls `glfwGetPlatform`, a symbol that exists only in GLFW
3.4:

```console
$ nm -u .../lib/libcimgui.dylib | grep glfwGetPlatform
_glfwGetPlatform
$ otool -L .../lib/libcimgui.dylib | grep -i glfw
	@rpath/libglfw.3.dylib (compatibility version 3.0.0, current version 3.4.0)
```

Resolve `GLFW_jll` to 3.3.9 and that symbol is missing, so the call
lands on a null pointer:

```
signal 11 (2): Segmentation fault: 11
unknown function (ip: 0x0) at (unknown file)
```

which is exactly what killed both `CImGui → GlfwOpenGLBackend`'s
precompile workload and every demo at runtime.

GLFW_jll cannot be raised to 3.4 while HarfBuzz is in the environment:

```
Unsatisfiable requirements detected for package Pango_jll:
 ├─restricted by compatibility requirements with HarfBuzz_jll to versions: 1.42.4
 └─restricted by compatibility requirements with libdecor_jll to versions: 1.52.2 - 1.58.0
```

The chain is: GLFW_jll 3.4 → libdecor_jll → Pango_jll ≥ 1.52, while
`HarfBuzz_jll` 100.x is accepted by no Pango_jll at all — every
Pango_jll from 1.47 on pins `HarfBuzz_jll` to the 2.x or 8.x series, and
only the ancient 1.42.4 (which predates compat entries) takes anything.

And HarfBuzz cannot simply be dropped: `ManyUICImGuiCImGuiExt` lists it
among its extension triggers, so without it the backend never loads.

**The fix is upstream and small.** The bound exists because the fork
needs `hb_draw_funcs_create`, which is documented in
`ManyUIDemos/Project.toml` as requiring HarfBuzz_jll 100.x. That is not
correct — `HarfBuzz_jll` 8.5.1 exports it:

```console
$ nm -gU .../libharfbuzz.0.dylib | grep -E '_hb_draw_funcs_create$'
0000000000012345 T _hb_draw_funcs_create
```

Relaxing the bound to `HarfBuzz_jll = "8.5.1, 100.14002"` resolves
cleanly — GLFW_jll 3.4.1, HarfBuzz_jll 8.5.1, Pango_jll 1.58.0 — and the
demos then start and render.

One symbol in `HarfBuzz.jl` is genuinely 100.x-only and would need a
guard: `hb_font_is_synthetic` (added in HarfBuzz 10). The subset API it
also calls lives in `libharfbuzz-subset`, which 8.5.1 does ship.
