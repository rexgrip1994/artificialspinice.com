# Citation map performance, 2026-10-01

The owner reported that zooming on the map lags. This report profiles `index.html`, finds where the time goes, and measures a patch that fixes most of it. The patch is `reports/map-performance-2026-10-01.patch`, a unified diff against main's `index.html` that contains only the performance changes. It applies with `git apply` and is meant to be ported to the private source.

## Method

- Setup: this checkout served with `python3 -m http.server`-style static hosting, headless Chromium (Playwright, 1440x900 viewport), `#map` opened and the layout left to settle (`sim.alpha() < alphaMin`). A seeded `Math.random` makes layouts reproducible for the visual comparison.
- Scenarios, run in Network, "By year" and Local graph (the highest-degree paper):
  - `zoom`: 10 wheel notches in, then 10 out, at 25 ms spacing.
  - `deepzoom`: 22 notches in, then 22 out.
  - `pan`: a drag of 50 mouse moves started on empty canvas.
  - `hover`: a sweep over about 60 nodes.
  - `zoom-while-sim`: `runSim(0.9)` followed by a zoom while the simulation is running.
- Frame time is the delta between consecutive `requestAnimationFrame` callbacks, recorded by a continuous rAF loop for the duration of each scenario. Dropped frames are the number of 16.7 ms slots missed (`floor(delta/16.7) - 1` summed). Medians are bounded below by 16.7 ms, the 60 Hz tick.
- Configurations: deviceScaleFactor 1 with CPU throttle 1x and 4x (`Emulation.setCPUThrottlingRate`), and deviceScaleFactor 2 with 1x.
  - **dpr 2 with 4x throttle was not run.** The container was restarted mid-run and dpr 2 frames already take 0.5 to 1 s without throttling.
  - **dpr 2 / 1x covers only the scenarios listed in its table below.** The remaining rows were not collected.
- Profiling: a CDP trace (`devtools.timeline,cc,viz,gpu,v8`) of the Network zoom, plus a phase breakdown. The phase breakdown forces a canvas flush (`getImageData`) after each phase of `drawGraph` in an instrumented copy, so recording cost and raster cost are separated.
- Visual check: screenshots of base and patched pages, deterministic layout, at fit, hover, zoomed in and zoomed out.
- **Caveat:** headless Chromium here rasterises canvas in software (no GPU). Absolute numbers are therefore pessimistic, especially at dpr 2. The ranking of where time goes should hold on a GPU, but the size of the gain there is unmeasured.

## Where the time goes

1. **The JavaScript side of a frame is cheap.** In the trace of 20 zoom frames, `FireAnimationFrame` totals 124 ms, about 6 ms per frame. `drawMain` measures 4 to 8 ms of JS per call. d3 work, label measurement and GC are all negligible.
2. **The cost is canvas raster at commit.** The same trace shows `Commit` / `LayerTreeHost::DoUpdateLayers` at 1.9 s for 20 frames, about 95 ms per frame. Frame time is 50 to 250 ms while the JS is 5 ms.
3. **Citation edges are almost all of it.** Per-frame flush cost by phase, in ms, at dpr 1:

| View | background + axis | citation edges | thesis dashes | nodes | labels |
|---|---|---|---|---|---|
| Network, fit | 1.4 | **28.0** | 1.4 | 1.7 | 0.04 |
| Network, x2 | 1.0 | **69.5** | 2.2 | 2.0 | 0.4 |
| Network, x5 | 0.8 | **80.7** | 1.1 | 0.6 | 0.3 |
| By year, fit | 1.3 | **33.4** | 1.4 | 1.7 | 0.04 |
| By year, x3 | 1.2 | **106.6** | 1.3 | 1.2 | 0.5 |

   At dpr 2 the edge flush is 790 to 1100 ms per frame, and every other phase is under 32 ms. Edge cost grows when zoomed in, because about 14,700 long hairlines are still rasterised, including their long off-screen tails.
4. **Simulation ticks during zoom.** Each tick bumps the frame to a full edge redraw, so zooming while the simulation runs cannot reuse any cached edges. rAF already coalesces tick-driven and zoom-driven redraws into one frame, so skipping the tick redraws would gain nothing. The tick computation itself is a few ms.
5. **Text labels, node paths, fills, dashes, layout and GC** are each at most about 2 ms per frame. They are not worth touching.

## Changes

Both changes are in the patch. Measured results are in the tables below.

1. **Clip every citation line to the visible rectangle and stroke in screen space.**
   - The previous code stroked world-space segments with a 0.6/k line width and culled only lines entirely off-screen. Segments that were partly visible were rasterised in full, including their off-screen tails.
   - Now each segment is Liang-Barsky clipped to the viewport plus a 10 px margin.
   - Output is pixel-identical to the original: max difference 2/255 and mean 0.001 in the screenshot comparison, with the edge cache turned off.
   - This applies in every frame where the cache is not used, such as while the simulation is running.
2. **Offscreen bitmap of the static edge layer.**
   - The edges are rendered once into an offscreen canvas covering the viewport plus a 30% margin, and that bitmap is blitted on later frames. It is reused when:
     - only the hover or selection changed (a 1:1 blit; dimmed edges use `globalAlpha` 0.35, equal to 0.035/0.10);
     - the view is zooming or panning (the bitmap is scaled and translated while it still covers the viewport and the scale ratio is within 0.55 to 2).
   - It is rebuilt exactly when the zoom or pan settles, 180 ms after the last zoom event, or when coverage is lost.
   - An `edgeEpoch` counter is incremented on every simulation tick, every `computeVis`, and every REDUCED-motion drag, so the bitmap is never stale. While the simulation is running the bitmap is not used.
   - The bitmap is freed when leaving the map view. With the pixel cap at 16 Mpx it is about 13 MB at dpr 1, and the margin is dropped when the 16 Mpx cap would be exceeded.
   - **Trade-off:** during an active zoom the edges are a resampled bitmap. They are slightly soft and snap to crisp when the gesture ends.
   - **Trade-off:** the cached layer is 8-bit premultiplied, so it differs a little from direct stroking. Edge-dense areas are about 1 to 3 levels (of 255) brighter. The mean over the whole zoomed view is +0.8/255, and a faint region at fit view is identical. This is not visible to the eye, but it is not bit-identical.
   - **Trade-off:** extra memory for the bitmap, as above.

Not done, with reasons:
- **Batching nodes by colour.** It changes the z-order of overlapping nodes, and node raster is at most 2 ms per frame.
- **Label caching and LOD.** Labels cost at most 1 ms and are already capped at 200 per frame.
- **Pausing simulation ticks during a zoom.** It would change the layout dynamics, and the redraws are already coalesced.
- **Subsampling edges at low zoom while the simulation runs.** This would change the look, and it is the obvious next step for `zoom-while-sim`.

## Results

Frame times in ms (rAF delta), and dropped 16.7 ms slots, per scenario.


#### deviceScaleFactor 1, CPU throttle 1x

| Scenario | Before median | Before p95 | Before dropped | After median | After p95 | After dropped |
|---|---|---|---|---|---|---|
| net/zoom | 49.9 | 150.0 | 138 | 16.7 | 83.3 | 33 |
| net/deepzoom | 16.7 | 100.1 | 93 | 16.7 | 33.3 | 31 |
| net/pan | 16.7 | 16.7 | 0 | 16.7 | 16.7 | 0 |
| net/hover | 16.7 | 16.8 | 6 | 16.7 | 16.7 | 0 |
| net/zoom-while-sim | 83.3 | 150.0 | 248 | 50.0 | 83.4 | 147 |
| year/zoom | 66.7 | 166.6 | 260 | 33.4 | 116.6 | 114 |
| year/deepzoom | 16.7 | 66.7 | 50 | 16.7 | 16.8 | 25 |
| year/pan | 16.7 | 16.7 | 0 | 16.7 | 16.8 | 1 |
| year/hover | 16.7 | 16.7 | 1 | 16.7 | 16.8 | 0 |
| year/zoom-while-sim | 16.7 | 33.4 | 26 | 16.7 | 33.3 | 7 |
| local/zoom | 66.6 | 166.6 | 222 | 50.0 | 150.0 | 173 |
| local/deepzoom | 16.7 | 100.0 | 91 | 16.7 | 50.0 | 45 |
| local/pan | 16.7 | 16.8 | 0 | 16.7 | 16.7 | 0 |
| local/hover | 16.7 | 16.7 | 0 | 16.7 | 16.7 | 0 |
| local/zoom-while-sim | 16.7 | 33.3 | 10 | 16.7 | 33.3 | 10 |

#### deviceScaleFactor 1, CPU throttle 4x

| Scenario | Before median | Before p95 | Before dropped | After median | After p95 | After dropped |
|---|---|---|---|---|---|---|
| net/zoom | 450.1 | 1133.4 | 1327 | 25.1 | 550.0 | 436 |
| net/deepzoom | 16.7 | 883.3 | 777 | 16.7 | 316.7 | 275 |
| net/pan | 16.7 | 50.1 | 102 | 16.7 | 16.8 | 10 |
| net/hover | 66.6 | 83.3 | 167 | 33.3 | 50.0 | 55 |
| net/zoom-while-sim | 650.0 | 1100.0 | 1696 | 250.0 | 433.3 | 665 |
| year/zoom | 466.7 | 866.6 | 1262 | 216.7 | 366.7 | 674 |
| year/deepzoom | 41.7 | 700.0 | 1036 | 16.7 | 416.6 | 643 |
| year/pan | 16.7 | 50.0 | 80 | 16.7 | 16.8 | 8 |
| year/hover | 66.6 | 83.4 | 164 | 49.9 | 50.1 | 94 |
| year/zoom-while-sim | 83.4 | 133.3 | 241 | 66.8 | 133.4 | 193 |
| local/zoom | 450.0 | 883.3 | 1290 | 316.7 | 616.7 | 927 |
| local/deepzoom | 16.7 | 999.9 | 1162 | 16.7 | 516.7 | 717 |
| local/pan | 16.7 | 16.8 | 4 | 16.7 | 16.7 | 1 |
| local/hover | 50.0 | 66.6 | 117 | 16.8 | 33.4 | 36 |
| local/zoom-while-sim | 100.0 | 150.0 | 202 | 66.5 | 116.7 | 106 |

#### deviceScaleFactor 2, CPU throttle 1x

| Scenario | Before median | Before p95 | Before dropped | After median | After p95 | After dropped |
|---|---|---|---|---|---|---|
| net/zoom | 1166.6 | 1650.0 | 4014 | 858.3 | 1200.1 | 2912 |
| net/deepzoom | 16.7 | 3699.9 | 4916 | 16.7 | 1166.6 | 1415 |
| net/pan | 16.7 | 583.4 | 1616 | 16.7 | 16.7 | 26 |
| net/hover | 550.0 | 649.9 | 1932 | 16.7 | 33.4 | 57 |
| net/zoom-while-sim | 1150.0 | 1866.6 | 4447 | 900.1 | 1183.2 | 3147 |
| year/zoom | 1233.3 | 2466.5 | 4993 | 675.0 | 916.6 | 2442 |
| year/deepzoom | 633.3 | 3016.6 | 4621 | 16.7 | 849.9 | 1132 |
| year/pan | 16.7 | 16.7 | 27 | 16.7 | 16.8 | 10 |
| year/hover | 483.4 | 550.0 | 1684 | 33.3 | 33.4 | 66 |
| year/zoom-while-sim | 91.6 | 649.9 | 778 | 16.8 | 466.7 | 468 |
| local/zoom | 733.3 | 966.6 | 2636 | 908.2 | 1966.5 | 3869 |
| local/deepzoom | 16.7 | 816.6 | 1565 | 16.7 | 750.0 | 1918 |
| local/pan | 16.7 | 16.7 | 40 | 16.7 | 16.7 | 18 |
| local/hover | 366.6 | 416.6 | 1229 | 16.7 | 16.8 | 0 |
| local/zoom-while-sim | 583.2 | 750.0 | 1523 | 416.6 | 616.7 | 1204 |


Reading the results:
- **Zoom.** The Network zoom median at dpr 1 / 1x falls from 49.9 ms to 16.7 ms. At 4x throttle it falls from 450 ms to 25 ms. At dpr 2 the median falls from 1.17 s to 0.86 s, and deep-zoom p95 falls from 3.7 s to 1.2 s.
- **Hover and pan.** These were jank-free at dpr 1 / 1x already. At 4x throttle they roughly halve or better. At dpr 2, hover goes from 550 ms to 16.7 ms and pan p95 from 583 ms to 16.7 ms.
- **Remaining slow cases.**
  - **Zoom while the simulation runs.** Every frame must redraw the clipped edges because positions change, so the gain here comes only from change 1: 83 ms falls to 50 ms at dpr 1 / 1x.
  - **First frame of a gesture, and the settle frame.** Each pays one full edge render. This is what the remaining p95 spikes are, and it is why the dpr 2 zoom is still slow in software raster.
  - **Local graph zoom.** It still shows about 50 ms median at dpr 1 / 1x. The simulation is still running for part of that scenario, and the cause was not isolated.
- **Noise.** One run per cell, on a shared 4-core container. Treat differences under about 20% as noise.

## Porting

`git apply reports/map-performance-2026-10-01.patch` against main's `index.html`. The patch adds `edgeEpoch`, `strokeEdges`, `buildEdgeCache` and `blitEdges`. It changes the edge block of `drawGraph`, the tick handler, `computeVis`, the zoom handler, the REDUCED-mode drag and `show()`. No visual options were added or removed. The benchmark scripts are not included.
