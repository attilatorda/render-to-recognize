# Render to Recognize

**A 3D vision pipeline with no meshes and no training loop.** Objects are drawn as pure distance-field equations, then recognized by rendering candidates and keeping the one that matches. Cryo-EM reads a molecule's orientation from a noisy image the same way. Everything runs client-side in raw WebGL, with no dependencies.

**[▶ Live demo](https://attilatorda.github.io/render-to-recognize/)**

---

## How it works

Everything follows from the first step.

1. **Draw without vertices.** A sphere is one short equation instead of a few thousand triangles. Because the surface is an equation, zooming in produces new detail rather than running out of it: the renderer adds [fBm](https://en.wikipedia.org/wiki/Fractional_Brownian_motion) octaves as the pixels get smaller — the same [level-of-detail](<https://en.wikipedia.org/wiki/Level_of_detail_(computer_graphics)>) trade a game engine makes with meshes — and [supersamples](https://en.wikipedia.org/wiki/Supersampling) the silhouette.
2. **Recognize by re-drawing.** To identify a hidden object, draw every candidate at every orientation and keep whichever drawing matches the observation. One search returns the object *and* how it is turned. Nothing is trained — the recognition lab has a step-by-step account of why that is possible, and of what gets paid instead. "Every orientation" means the whole [rotation group](https://en.wikipedia.org/wiki/3D_rotation_group), not a ring of spins about one axis, and that is what makes the search expensive: SO(3) is three-dimensional, so a grid three times finer costs 27 times the poses.
3. **Make it fast.** Draw the candidates once and store each view as a short list of numbers; recognition is then a [nearest neighbor search](https://en.wikipedia.org/wiki/Nearest_neighbor_search) with no drawing at query time. Symmetric shapes compress hard, because many of their rotations look identical. Single-particle microscopy assigns orientations this way.

**Getting the shape right and the pose wrong is its own answer.** Both recognition demos say so in yellow rather than showing a green tick, and both work out which poses are confusable by measuring it instead of asserting it: the recognition lab sweeps its pose grid for views that render alike but sit far apart in SO(3) (sphere 100%, torus 95%, holed cube 13%, chair 0%), and the codebook lab records what each stored view swallowed when it de-duplicated (median 162° for a filament, 0° for a helix) and flags entries with a look-alike in a *different* structure. When two answers really do fit, both are shown.

## The demos

| File | What it shows |
| --- | --- |
| `index.html` | Project homepage: live hero, the narrative, and links to the demos. Start here. |
| `sdf-sculpt-demo.html` | Vertexless renderer. One equation-only surface; orbit it, carve procedural detail, blend in a second blob. Scroll or pinch to magnify up to 64×. View normals and depth as free G-buffer channels. |
| `sdf-recognition-lab.html` | Analysis by synthesis. A hidden object at an orientation drawn uniformly from SO(3) is blurred and occluded; the lab scores all five candidates over 2,560 orientations each, refines the winner, and reports the match, its orientation, and a confidence ranking. Colour is display-only and never reaches the matcher. |
| `micrograph-codebook-lab.html` | The codebook speedup, reskinned as microscopy. Build a de-duplicated view dictionary over all of SO(3) once, then identify noisy "micrographs" by lookup. An axially symmetric filament collapses to a handful of views where the helix keeps hundreds. |

## Methods

None of these techniques is new. The work is in building them end to end and making them run in a browser.

- **Rendering.** [Signed distance functions](https://en.wikipedia.org/wiki/Signed_distance_function) and [sphere tracing](https://en.wikipedia.org/wiki/Ray_marching) (Hart, *Sphere Tracing*, 1996; see also Inigo Quilez's articles at iquilezles.org). The holed cube is a [constructive solid geometry](https://en.wikipedia.org/wiki/Constructive_solid_geometry) subtract; edges use rotated-grid [supersampling](https://en.wikipedia.org/wiki/Spatial_anti-aliasing).
- **Recognition.** Render-and-compare, or [template matching](https://en.wikipedia.org/wiki/Template_matching) against synthesized candidates, which solves [3D pose estimation](https://en.wikipedia.org/wiki/3D_pose_estimation) at the same time. It is the backbone of learned refiners like DeepIM (Li et al., 2018) and CosyPose (Labbé et al., 2020). The pose search is coarse-to-fine: a grid over SO(3), then local refinement around the best few cells. Refining only the single best cell is not enough, because at a grid spacing of 22° a chair keeps just 42% of its peak score and the right basin is often not ranked first.
- **In-plane rotation is free.** Rolling the camera about its own view axis spins the rendered image and changes nothing else, so one render per view direction is scored against every roll by spinning the observation instead. The recognition lab covers 2,560 orientations per shape with 160 renders. Projection matching in cryo-EM does the same thing for the same reason.
- **Colour is never evidence.** Objects are drawn in a colour the viewer picks or randomizes, but every buffer the matcher compares is rendered in one neutral grey shared by all shapes, and hues are applied afterwards when a buffer is painted. Verified, not assumed: changing every colour leaves the offscreen renders byte-identical.
- **Codebook.** The **augmented autoencoder** approach to pose estimation (Sundermeyer et al., *Implicit 3D Orientation Learning*, ECCV 2018), where views are embedded once and recognition is a lookup. Novelty-based de-duplication makes it a small [vector quantization](https://en.wikipedia.org/wiki/Vector_quantization) of the view sphere.
- **Microscopy framing.** Projection matching, how cryo-EM software (RELION, cryoSPARC) assigns particle orientations in [single particle analysis](https://en.wikipedia.org/wiki/Single_particle_analysis) by comparing images against projections of a 3D map.

## Scope

Render-and-compare only works when you can draw the thing you are looking for. That covers rigid objects with a known shape — molecular machines, viral capsids, organelle morphology — and it recovers identity and orientation in the same search.

It does not cover things that differ from one sample to the next. Cells on a pathology slide have no fixed template to render against, which is why real cancer detection is supervised classification and not this.

Two limits worth stating outright: the biological shapes here are stylized stand-ins rather than accurate molecular geometry, and none of the methods are new. The work is building them end to end and measuring what they do.

## Built with

Raw WebGL and GLSL fragment shaders do the ray marching, vanilla JavaScript drives them, and Canvas 2D draws the image panels. Each demo is a single self-contained HTML file, so a browser is the only thing you need to run one.

## Run it

It is static, so no server is needed for the demos themselves, though a local server avoids browser file-path quirks:

```bash
# clone, then from the project folder:
python3 -m http.server 8000
# open http://localhost:8000
```

**Deploy free:** push to a GitHub repo and enable **Settings → Pages → deploy from branch**, or drag the folder onto [Netlify Drop](https://app.netlify.com/drop). Keep all files in one folder so the homepage links resolve.

## Where it could go next

- **Learned embedding.** Replace the hand-made 10×10 vector with a small TensorFlow.js autoencoder trained on the renders. That is the real augmented autoencoder, and these demos already generate perfectly labeled training data.
- **Real data.** The only realistic path to a publishable result would be running the codebook against a public cryo-EM dataset (EMPIAR, say) with a proper baseline.

## Further reading

Three references cover the ground behind this project, and all are free from their authors:

- **Computer Vision: Algorithms and Applications** (Szeliski). The standard reference; covers analysis-by-synthesis and pose estimation. Free draft: [szeliski.org/Book](https://szeliski.org/Book)
- **Physically Based Rendering** (Pharr, Jakob, Humphreys). The definitive text on how production renderers actually work, and why film path-traces meshes instead of ray-marching equations. Free: [pbr-book.org/4ed](https://pbr-book.org/4ed)
- **SDFs and ray marching.** There is no single canonical book here; the reference material is Inigo Quilez's articles at [iquilezles.org/articles](https://iquilezles.org/articles).

Worth buying used, where older editions are cheap: *Real-Time Rendering* (Akenine-Möller et al.) and *Multiple View Geometry in Computer Vision* (Hartley & Zisserman).

## Credits

Built by Attila Torda. [github.com/attilatorda](https://github.com/attilatorda)

Source: [github.com/attilatorda/render-to-recognize](https://github.com/attilatorda/render-to-recognize)

## License

**Proprietary, all rights reserved.** This code is published for viewing and evaluation only, as a portfolio piece. It may not be used, copied, modified, redistributed, or used commercially without the author's prior written permission. See [`LICENSE`](LICENSE) for the full terms.
