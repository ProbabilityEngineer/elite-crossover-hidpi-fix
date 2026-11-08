# Elite Dangerous CrossOver HiDPI Fix

A small, reversible **per-game registry workaround** for half-resolution output
in Elite Dangerous on CrossOver.

Use it together with the bottle's **High Resolution Mode**. It does not modify
the game executable, install a DLL, or require ReShade.

## The problem

On the tested Retina setup, Elite could render its scene at **3024×1964** while
its output/back buffer was only **1512×982**. The resulting image looked soft,
even though the game was doing the work of rendering a larger scene.

Selecting a higher resolution or changing between fullscreen and borderless
did not reliably keep native-sized output. One recorded transition successfully
requested a larger back buffer, then replaced it with 1512×982 just 38 ms later.
Increasing supersampling could increase GPU work and memory use without fixing
the smaller final output.

The suspected cause is **DPI virtualization**: a game that is not treated as
DPI-aware can receive logical window/desktop dimensions rather than the pixel
dimensions expected for native-resolution rendering.

The tested Elite executable's embedded manifest did not declare DPI awareness.
That alone does not establish its effective runtime awareness, but applying the
override below with High Resolution Mode enabled produced sustained
native-sized output in the subsequent test.

## What the fix changes

[`elite-crossover-hidpi-fix.reg`](elite-crossover-hidpi-fix.reg) sets one string
value inside the selected bottle:

```reg
Windows Registry Editor Version 5.00

[HKEY_CURRENT_USER\Software\Microsoft\Windows NT\CurrentVersion\AppCompatFlags\Layers]
"elitedangerous64.exe"="HIGHDPIAWARE"
```

The Wine implementation inspected for this workaround interprets
`HIGHDPIAWARE` as **system-DPI aware**, during process initialization.

**Wine-specific detail:** that implementation looks up the executable's
lowercase filename, `elitedangerous64.exe`, rather than its full Windows path.
This is not a general-purpose native Windows registry recipe.

The override:

- Applies to that executable name in that bottle and Windows user account.
- Does not change the bottle-wide default compatibility flags.
- Does not set the game's resolution, supersampling, AA or HDR settings.
- Does not itself enable CrossOver's High Resolution Mode.

## Before importing: preserve existing flags

**Importing this file replaces the named value; it does not append a flag.**

In Registry Editor, check whether `elitedangerous64.exe` already exists under
the key above. If it does:

1. Back up the existing value or export the selected registry key.
2. Do not import this file unchanged.
3. Instead, edit the existing string and add `HIGHDPIAWARE`, separated from other
   flags by a space. Resolve any conflicting DPI flags deliberately rather than
   blindly retaining both `HIGHDPIAWARE` and `DPIUNAWARE`.

Do **not** put this flag in the key's `(Default)` value. Wine can use that as a
default for other applications.

## How to apply

1. Download `elite-crossover-hidpi-fix.reg`.
2. Quit Elite normally.
3. In CrossOver, select **the bottle containing Elite**.
4. Enable **High Resolution Mode** in that bottle's advanced settings.
5. Open **Registry Editor** for that same bottle. If necessary, use CrossOver's
   **Run Command** and run `regedit`.
6. Check for an existing value as described above.
7. In Registry Editor, choose **File → Import**, select
   `elite-crossover-hidpi-fix.reg`, and confirm the import.
   - Wine's `Z:` drive normally exposes the Mac filesystem. For example, a
     downloaded file may be under `Z:\Users\<your-name>\Downloads\`.
   - Do not rely on double-clicking the file in Finder: it may open a different
     bottle.
8. Verify that the key contains:

   ```text
   elitedangerous64.exe    REG_SZ    HIGHDPIAWARE
   ```

9. Quit the bottle's applications normally and restart the bottle before
   relaunching Elite, so the HiDPI setting and startup override take effect.

For the initial check, select your display's native pixel resolution and use
**supersampling 1.0×**. Do not compensate for a small output buffer by raising
supersampling.

## How to check whether it worked

Check **output resolution**, not just the game's selected resolution or its
internal scene-render size.

On the tested 3024×1964 display, the desired output is **3024×1964**, not
1512×982. If the Metal HUD is available, check its resolution too.

If using EDVR, its route reports distinguish:

- `R`: scene render dimensions.
- `E`: temporal backend output dimensions.
- `D`: display/output dimensions observed by EDVR.

The successful test reported:

```text
trained-upscale R=2268x1473 E=3024x1964 D=3024x1964
trained-native R=3024x1964 E=3024x1964 D=3024x1964
```

Both have native-sized output. The first intentionally renders a smaller scene
and upscales it; the second renders and outputs at native resolution. There was
no logged return to half-sized output through the inspected interval.

Those logs verify the DXGI/output dimensions, not an independent measurement
of every stage of macOS presentation.

If output still falls back to half resolution, the workaround has not been
verified on your setup. Check the bottle, the named registry value, the
High Resolution Mode switch and whether Elite was fully restarted. Do not
assume a larger supersampling setting proves the fix worked.

## Undo

Quit Elite and open Registry Editor in the same bottle.

- If the named value did **not** exist before: delete only the
  `elitedangerous64.exe` value.
- If it already existed: restore its previous contents from your backup.

Do not delete the entire `Layers` key or another application's values.
Restart Elite afterward. The bottle's High Resolution Mode setting is separate;
restore its previous setting too if you want to undo the entire experiment.

## Tested configuration and limitations

Observed on 8 November 2026 with:

- Elite Dangerous: Odyssey, DX11.
- Apple M4 Max / 36 GB unified memory / 3024×1964 Retina display.
- CrossOver Preview: bundle build `27.0.0.41073`
  (`CFBundleShortVersionString`: `20261015`).
- DXMT [CI run 42205734568](https://github.com/3Shain/dxmt/actions/runs/42205734568),
  merge commit `86c378136df6fc000df697da1d95fe78e726c699`.
- EDVR `v0.22.0-rc.7`, build `6AED3509`, present during validation and used to
  report rendering/output dimensions.

This is a configuration workaround, **not a patch to EDVR, DXMT or Wine**.
EDVR is not required to apply the registry change, but a separate test without
EDVR has not been established here. Other CrossOver versions, graphics backends,
display configurations and DPI-related side effects have not been comprehensively
tested.

Native-resolution output can cost more GPU time and memory than half-resolution
output. This is not an FPS boost or a memory-leak fix.
