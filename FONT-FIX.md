# Cross-platform font fix (o1laabs fork)

This fork patches a bug that makes upstream unusable outside macOS.

## The bug

`scripts/render_animated_diagram.py` → `font_candidates()` listed **only** macOS system fonts:

```python
"/System/Library/Fonts/Supplemental/Chalkduster.ttf"
"/System/Library/Fonts/STHeiti Light.ttc"
"/System/Library/Fonts/Helvetica.ttc"
...
```

On Linux and Windows every path raises `OSError`, and `load_font()` swallowed the
error and returned Pillow's built-in bitmap font:

```python
return ImageFont.load_default()
```

That default is a **fixed 10px bitmap font that ignores the `size` argument**:

```
load_font(28, cjk=True) -> ('Aileron', 'Regular') size 10
```

Because every size candidate then measures identically, `fit_text()` returns the
first candidate immediately — so **text never wraps, never scales down, and CJK
renders as tofu**.

Upstream's own suite fails **4 of 7 tests** on Linux, all in text fitting
(`test_wraps_cjk_text_without_spaces`, `test_wraps_english_text_to_width`, ...),
because they assert `assertIn("\n", text)` and no wrapping ever happens.

## What this fork changes

1. **`font_candidates()`** now returns macOS paths first (upstream behaviour is
   preserved on Mac), then Linux distro paths, then Windows:

   | Platform | Fonts tried |
   |---|---|
   | macOS | Chalkduster, STHeiti, Helvetica, Arial Unicode, PingFang |
   | Linux | Noto Sans CJK (Alpine/Debian/Fedora paths), WenQuanYi Zen Hei, Noto Sans, DejaVu Sans, Liberation Sans |
   | Windows | `C:\Windows\Fonts` — Segoe UI, Arial, Microsoft YaHei, SimHei |

2. **`load_font()`** gained a `fc-match` (fontconfig) last resort and now
   **warns on stderr** instead of failing silently when no scalable font is found.

3. **Tests** load the renderer from `scripts/` and `assets/` from the skill root,
   plus a `tests/__init__.py` so `unittest discover` works.

## Verification

Environment: Alpine aarch64 (PRoot), Python 3.12, Pillow 12.3.0.

```
$ python3 -m unittest discover -s tests -t . -v
Ran 7 tests in 23.1s
OK                      # was: FAILED (failures=4)
```

```
$ python3 scripts/render_animated_diagram.py \
    --spec assets/default-spec.json \
    --outdir outputs --basename lanshu --verify --check
elements: 159
frames:   41
diffs:    (0,10,173488) (10,20,125440) (20,30,186821) (30,40,193245)
checks ok: True  (13/13)
```

Rendered PNG grew from 170 KB (nothing drawn) to 344 KB (text actually drawn).

Font resolution after the fix:

```
cjk         -> ('Noto Sans CJK JP', 'Regular') size=32
cjk bold    -> ('Noto Sans CJK JP', 'Bold')    size=32
latin       -> ('Open Sans', 'Regular')        size=32
latin bold  -> ('Open Sans', 'Bold')           size=32
hand        -> ('Open Sans', 'Regular')        size=32   # no Linux equivalent
fit_text("研究问题收敛与证据综合", 90, 44, 16) -> ('研究问题收敛\n与证据综合', 15)
```

## Caveats

- **`hand=True` has no true cross-platform equivalent.** Chalkduster and
  MarkerFelt are macOS-only. On Linux the output is clean but not hand-drawn —
  the "岚叔风" visual signature is macOS-bound.
- **`--check` passing does not mean the diagram is correct.** It validates output
  structure (dimensions, `fontFamily: 5`, empty `files`), and it passed green
  even when not a single glyph was drawn. Always open the PNG.
- Upstream README claims macOS only in passing; the failure is silent, which is
  what makes it worth fixing rather than documenting.

Upstream: <https://github.com/cclank/lanshu-animated-architecture-diagram> (MIT)
