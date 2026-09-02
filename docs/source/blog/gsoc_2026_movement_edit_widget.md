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

In multi-animal tracking, automated pose-estimation pipelines routinely fail because of occlusions and identity ambiguity. The result is data with misplaced keypoints, missing detections, and individuals whose identities get swapped from one frame to the next. Before these tracks can be used for analysis, a researcher has to go through the recording and fix the errors either programmatically or by hand. However, tools available for proofreading/refinement are usually very slow, unintuitive and hard to scale to large datasets. 

This projects fills this gap inside the `movement` ecosystem by building an edit-and-save widget for its napari GUI. Concretely, the widget lets users drag, add and remove keypoints with real-time updates, correct identity swaps across frames and save the edited dataset straight back to `movement`'s native format. The editing itself relies on native `napari` tools, so anyone familiar with napari can get started quickly. Every change a user makes is flagged in the data and the GUI, so refined points stay easy to identify afterwards. 


## Technical Implementation

### From dataset to layers, and back

Loading a dataset in `movement` napari's GUI means flattening `movement`'s `(individuals, keypoints, space, time)` array into the flat array of points that napari expects. Missing detections (`NaN`) are not drawn in the `Points` layer, so at any frame the layer only holds the points that actually exist. 

Before making any edit on the dataset, we needed to write the data back from napari layer's to `movement` `xarrays`. The function that converts `xarrays` to `napari` layers is `ds_to_napari_layers`. We implemented a mirror function named `napari_layers_to_ds` that deals with the way back trip: reverses this flattening process, rebuilding the full `(individuals, keypoints, space, time)` array from the layer and its properties, and re-inserts `NaN` everywhere a point is missing. 

Later work extended this function to handle edits. Because the position array is rebuilt from the live layer, a dragged point needs no special treatment: its new coordinates are already there. A deleted point is trickier, since it leaves no row in the layer at all; the function compares the live layer against the original structure to tell a user deletion from an always-missing point, and restores both as `NaN`. A keypoint or individual left with no data anywhere is then dropped from the dataset, a frame from which every point was removed is kept and filled with `NaN`, and deleting every point raises an error rather than returning an empty dataset. 


### Tracking what the user changed

For manual curation to be useful, an edited point has to be distinguishable from an untouched one. We added an `edited` boolean to the properties table, set to `True` whenever a point is dragged or deleted, by listening to the `Points` layer's data-change events. 
When a point is moved, its confidence no longer means anything. The confidence comes from the model, not the user, so it is reset to `NaN` at the same time. Deletions are also edits, so removed points are flagged too; a `position_is_nan` property records which points held real data before the deletion, and is used to reconstruct the new dataset with removed points set as `True` in `edited` properties. 

### Showing the edits

Those flags then drive the display. Edited points are drawn as a hollow ring instead of a filled marker, using napari's per-point symbol support, and the ring survives a reload because it is derived from the `edited` property rather than set by hand. To see edits across a whole recording rather than one frame at a time, a timeline widget embeds a Matplotlib canvas in a dock at the bottom of the viewer and marks every edited frame with a blue bar, so a user can scan the recording and jump straight to the frames they have already touched.

<!-- Screenshot: filled vs. ring points; and the timeline widget with blue bars. -->

### Keeping the layers in sync

Because the `Points` and `Tracks` layers are two views of the same data, an edit to one has to propagate to the other. When a point is dragged or deleted, the corresponding vertex of the trajectory is updated in place, so the track line follows the correction immediately instead of only after the file is reloaded.

### Saving

The Save widget ties it together. It first checks that the `Points` layer was created by `movement`'s loader — editing an arbitrary napari layer and trying to save it as a pose dataset should fail cleanly — then runs `napari_layers_to_ds()` and writes the result to a NetCDF file through `movement.io`, so the corrected tracks come out in the same format they went in.

<!-- Screenshot: the Save panel. -->


## What I did

### Background

<!-- TODO: how you found the project, first steps in the codebase, mental map of the code -->

Before the coding period I contributed to several open-source projects to get comfortable with collaborative development — including [PlotlyBrain](https://github.com/), the `scipaper` classifier, and DeepLabCut ([PR #2715](https://github.com/DeepLabCut/DeepLabCut/pull/2715), adding function tests and improving docstrings).

### PRs created during the coding period

Over the coding period I opened the following PRs on `movement`. 

1. **Conversion from napari Points layers to movement pose datasets** ([#1011](https://github.com/neuroinformatics-unit/movement/pull/1011)) — added a `napari_layers_to_ds()` function that converts edited napari `Points` layers back into a `movement` pose dataset, rebuilding the original structure and restoring the missing (NaN) detections that were dropped when the layers were first created.

2. **Update bbox nan fixture** ([#1020](https://github.com/neuroinformatics-unit/movement/pull/1020)) — fixed the `valid_bboxes_dataset_with_nan` test fixture so that a missing detection is represented by NaN across the position, shape, and confidence arrays, matching how real bounding-box datasets behave.

3. **Set confidence of edited points in napari to NaN** ([#1024](https://github.com/neuroinformatics-unit/movement/pull/1024)) — when a user moves a point in napari, its confidence value is now reset to NaN, and `napari_layers_to_ds()` preserves that change when reconstructing the dataset.

4. **Reconstruction of removed napari pose predictions back to movement ds** ([#1025](https://github.com/neuroinformatics-unit/movement/pull/1025)) — extended `napari_layers_to_ds()` to handle deleted points: removed predictions come back as NaN, keypoints or individuals left with no data are dropped, and deleting every point raises an informative error.

5. **Adding edited properties to napari layers when a keypoint is dragged** ([#1041](https://github.com/neuroinformatics-unit/movement/pull/1041)) — added an `edited` boolean property on the napari layers, set to `True` when a point is dragged, so manually corrected keypoints can be told apart from originally missing ones.

6. **Save widget for napari plugin** ([#1044](https://github.com/neuroinformatics-unit/movement/pull/1044)) — added a "Save" widget that writes manually edited tracks back to disk: it checks the layer was created by `movement`'s loader, converts it to a `movement` dataset, and saves it as a NetCDF file.

7. **Changing the point symbol to ring for edited points in napari** ([#1053](https://github.com/neuroinformatics-unit/movement/pull/1053)) — edited keypoints are now drawn as a ring instead of a filled marker, based on their `edited` flag, and keep that appearance across dataset reloads.

8. **Synchronize tracks layer to points layer** ([#1054](https://github.com/neuroinformatics-unit/movement/pull/1054)) — the napari `Tracks` layer now updates immediately when a point is dragged, so the trajectory line follows the corrected position without needing a file reload.

9. **Set property edited = True on removed predictions** ([#1057](https://github.com/neuroinformatics-unit/movement/pull/1057)) — points are also flagged as `edited=True` when removed, not only when dragged, using a new `position_is_nan` property to detect which valid detections were deleted.

10. **Add napari timeline widget to flag edited points** ([#1063](https://github.com/neuroinformatics-unit/movement/pull/1063)) — added a docked timeline widget that shows blue bars on edited frames, making it easy to spot and jump to frames that have been manually corrected.

In addition to writing code, I reviewed the following PRs:

1. **Add `keep_points_with_nan_confidence` kwarg to `filter_by_confidence`** ([#1037](https://github.com/neuroinformatics-unit/movement/pull/1037)) — adds an option to `filter_by_confidence` that keeps points with NaN confidence (e.g. manually proof-read SLEAP annotations) instead of silently dropping them.

2. **Restore sync-tracks functionality dropped during PR #1054 rebase** ([#1083](https://github.com/neuroinformatics-unit/movement/pull/1083)) — recovers code lost in a rebase, restoring synchronisation of point deletions from the points layer to the tracks layer and the editing restrictions for non-default axis orders, together with their tests and docs.

3. **Update FOSDEM links to archive version** ([#1087](https://github.com/neuroinformatics-unit/movement/pull/1087)) — fixes CI linkcheck failures by pointing two FOSDEM schedule links at their `archive.fosdem.org` equivalents.


## Challenges

Writing the data back is the harder half, and most of my coding period went into it.


## Future work

GSoC gave the widget its foundations, but the full tool we designed in [issue #993](https://github.com/neuroinformatics-unit/movement/issues/993) is not finished yet. After the program I plan to keep contributing to `movement` to complete it: closing the remaining subissues tracked under that issue: [issue #996](https://github.com/neuroinformatics-unit/movement/issues/996), [issue #1006](https://github.com/neuroinformatics-unit/movement/issues/1006), [issue #998](https://github.com/neuroinformatics-unit/movement/issues/998), [issue #1009](https://github.com/neuroinformatics-unit/movement/issues/1009), [issue #1052](https://github.com/neuroinformatics-unit/movement/issues/1052), [issue #1059](https://github.com/neuroinformatics-unit/movement/issues/1059), [issue #1060](https://github.com/neuroinformatics-unit/movement/issues/1060)

Beyond that, I want to keep improving usability, respond to issues on the repository as people start using the tool, and keep the GUI components stable across releases. I will also join the fortnightly `movement` community calls to stay up to date with where the package is heading and make sure the widget evolves along with it.

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
- [My GitHub profile](https://github.com/anna-teruel)