:blogpost: true
:date: September 7, 2026
:author: Anna Teruel-Sanchis
:location: Bremen, Germany
:category: Blog
:language: English
:image: 0

# A napari GUI for manual tracklet refinement in `movement` - Final GSoC Report

## Introduction

Hi, I'm [Anna](https://github.com/anna-teruel). I'm a neuroscientist interested in the study of animal social behaviour. 
During my PhD I started using different open-source pose-estimation tools, such as [DeepLabCut](https://github.com/DeepLabCut/DeepLabCut), to track animal behaviour. Working with real, multi-animal recordings, I kept running into the same problems: occlusions, noisy detections, and identity swaps that no automated pipeline could fully resolve. Programmatic corrections improved the data, but visual inspection and manual correction were always necessary, and the tools available for that were slow and hard to scale to large datasets.

This summer, through [Google Summer of Code](https://summerofcode.withgoogle.com/) I had the opportunity to work with the [Neuroinformatics Unit](https://neuroinformatics.dev/) on [`movement`](https://movement.neuroinformatics.dev/), a Python toolbox for analysing animal body movements in pose-estimation data. My project added an interactive [napari](https://napari.org/)-based GUI for correcting pose-estimation tracks directly within the `movement` framework.

**Mentors:** [Sofía Miñano](https://github.com/sfmig), [Niko Sirmpilatze](https://github.com/niksirbi), [Chang Huan Lo](https://github.com/lochhh)

## Project Overview

In multi-animal tracking, automated pose-estimation pipelines routinely fail because of occlusions, image noise, and identity ambiguity. The result is data with misplaced keypoints, missing detections, and individuals whose identities get swapped from one frame to the next. Before these tracks can be used for analysis, a researcher has to go through the recording and fix the errors by hand.

`movement` already provides a standardized data structure for pose tracks and a growing set of analysis tools, but it had no way to *edit* tracks interactively. My project closed that gap by building a GUI, integrated with `movement` and napari, that lets users:

- inspect trajectories overlaid on the source video,
- move, add, and remove keypoints with real-time updates,
- correct identity swaps across frames,
- mask low-confidence detections, and
- interpolate short missing segments,

all while keeping the data in `movement`'s standardized format, so that corrected tracks can flow straight back into an analysis pipeline. Import and export support covers DeepLabCut, SLEAP, and `movement`'s own formats.

The project builds on an initial napari-based prototype for manual trajectory correction that I developed before GSoC.

## Technical Implementation

<!-- TODO: expand with the concrete design once the work is in.
     Points to cover:
     - How the widget is structured (napari plugin / dock widget) and which
       layers it uses: Points for keypoints, Tracks for trajectories, Image for video.
     - How edits are synchronised back to the movement `xarray` dataset in real time.
     - How individuals / keypoints / confidence are represented and toggled.
     - Import/export through movement.io (DeepLabCut, SLEAP, movement).
     - Performance work for large datasets (lazy loading, current-frame-window
       rendering, etc.).
     - Use of threads/workers to keep the UI responsive, if applicable. -->

**GUI framework:** The tool is built as a napari widget, reusing napari's `Points`, `Tracks`, and `Image` layers to display keypoints, trajectories, and the source video.

**Integration with `movement`:** Edits made in the GUI are written back to `movement`'s `xarray`-based data structure, so the in-memory dataset always matches what the user sees on screen.

**Import / export:** Pose data is loaded and saved through `movement.io`, supporting DeepLabCut, SLEAP, and `movement` formats.

## What I did

### Background

<!-- TODO: how you found the project, first steps in the codebase, mental map of the code -->

Before the coding period I contributed to several open-source projects to get comfortable with collaborative development — including [PlotlyBrain](https://github.com/), the `scipaper` classifier, and DeepLabCut ([PR #2715](https://github.com/DeepLabCut/DeepLabCut/pull/2715), adding function tests and improving docstrings).

### PRs created during the coding period

<!-- TODO: replace the placeholders below with your actual PRs, links and status.
     Grouped roughly by theme, following the plan from the proposal. -->

1. **Adapting the prototype widget to `movement` — PR [#XXX](https://github.com/neuroinformatics-unit/movement/pull/XXX)**
    - Ported the standalone napari prototype into a `movement` widget.
    - Improved trajectory visualisation and added multi-animal support.
    - **Status:** <!-- Merged / Open -->

2. **Real-time keypoint editing and synchronisation — PR [#XXX](https://github.com/neuroinformatics-unit/movement/pull/XXX)**
    - Implemented move / add / remove for keypoints with live updates to the underlying dataset.
    - **Status:** <!-- Merged / Open -->

3. **Identity correction and confidence masking — PR [#XXX](https://github.com/neuroinformatics-unit/movement/pull/XXX)**
    - Added manual correction of identity swaps across frames.
    - Added masking of low-confidence detections.
    - **Status:** <!-- Merged / Open -->

4. **Saving and exporting corrected data — PR [#XXX](https://github.com/neuroinformatics-unit/movement/pull/XXX)**
    - Export of corrected tracks to DeepLabCut, SLEAP, and `movement` formats.
    - **Status:** <!-- Merged / Open -->

5. **Interpolation of short missing segments — PR [#XXX](https://github.com/neuroinformatics-unit/movement/pull/XXX)**
    - **Status:** <!-- Merged / Open -->

6. **Performance optimisation for large datasets — PR [#XXX](https://github.com/neuroinformatics-unit/movement/pull/XXX)**
    - **Status:** <!-- Merged / Open -->

7. **Tests and documentation — PR [#XXX](https://github.com/neuroinformatics-unit/movement/pull/XXX)**
    - Unit tests for the editing logic and widget behaviour.
    - User-facing documentation and an example workflow.
    - **Status:** <!-- Merged / Open -->

<!-- TODO: mention any code review you did, with a link to a search of your reviewed PRs -->

### Screenshots

<!-- TODO: add screenshots of the widget.
     Save images under docs/source/_static/blog_images/movement_gsoc2026/
     and reference them as below. -->

```{image} /_static/blog_images/movement_gsoc2026/widget-overview.png
:align: center
:width: 80%
```
<p style="text-align:center;margin:8px;color:#9c9c9c;font-style:italic">The tracklet refinement widget in napari</p>

## Challenges

<!-- TODO: 2-3 concrete challenges and how you solved them. Candidates:
     - Keeping the GUI responsive while editing large datasets.
     - Mapping napari layer edits back onto movement's xarray structure without
       losing metadata (individuals, keypoints, confidence).
     - Designing an intuitive workflow for identity-swap correction.
     - Handling the different conventions of DeepLabCut / SLEAP on import/export. -->

## Future work

After GSoC I plan to keep contributing to `movement` by maintaining and extending the widget: improving usability, responding to user feedback (issues and bugs), and keeping the GUI components stable across releases.

Beyond this project, I'm interested in data validation and quality-control tools for `movement` — automatically identifying and flagging likely identity swaps, missing detections, or implausible movements — and in semi-automated correction suggestions that a user could accept or reject through the GUI, reducing manual workload while keeping the user in control of the final data.

## Conclusions

<!-- TODO: short wrap-up. What the tool enables for researchers, what you learned
     about collaborative open-source development and UX design, and how it felt to
     take something from prototype to a usable feature. -->

## Acknowledgements

I want to thank my mentors, [Sofía Miñano](https://github.com/sfmig), [Niko Sirmpilatze](https://github.com/niksirbi), and [Chang Huan Lo](https://github.com/lochhh), for their guidance throughout the project. I learned a lot from them. Especially about writing more efficient code, working as a team in software development, designing the core functions of an implementation, and testing everything thoroughly. These are methods I feel I can carry with me into future work in open-source, and I found the whole process genuinely inspiring. I'm also very grateful for the attention they gave to me and to the project. The regular communication has been fundamental to this work, and I'm thankful for that.

Thanks to [Adam Tyson](https://github.com/adamltyson), head of the Neuroinformatics Unit, for hosting the project and making this opportunity possible. And thank you for sending nice stickers and a `movement` mug! 

I would also like to thank [Juan Nunez-Iglesias](https://gist.github.com/jni) for generously sharing a snippet showing how to embed a Matplotlib canvas inside a napari dock widget, on `napari`'s zulip channel. It was both useful and inspiring for this project.

Finally, thank you to Google and the Google Summer of Code program for the opportunity, and for supporting new contributors as they take their first steps into open-source development.

## Related links

- [`movement` repository](https://github.com/neuroinformatics-unit/movement)
- [`movement` documentation](https://movement.neuroinformatics.dev/)
- [GSoC project proposal](<!-- TODO: link -->)
- [My GitHub profile](https://github.com/anna-teruel)