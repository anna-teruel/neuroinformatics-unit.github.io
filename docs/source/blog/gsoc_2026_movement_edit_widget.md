:blogpost: true
:date: September 7, 2026
:author: Anna Teruel-Sanchis
:location: Bremen, Germany
:category: Blog
:language: English
:image: 0

# A napari GUI for manual tracklet refinement in `movement` - Final GSoC Report

## Introduction

Hi, I'm [Anna](https://github.com/anna-teruel). I'm a neuroscientist interested in the study of social behaviour. 
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

Keep track of what has already been edited matters as much as making the edit. This widget shows edits in two places: (1) on the individual points of the current frame; and (2) on a timeline spanning the whole recording. 

On each individual frame, an edited keypoint is drawn as a hollow ring instead of a filled point. The symbol is derived directly from the `edited` property, so it is rebuilt every time the dataset is reloaded rather than being applied by hand. Scanning a single frame, the user can immediately tell edited (dragged) points from untouched ones. Points that are deleted are removed from the `Points` layer. 

<!-- Screenshot: the Points layer in a frame, showing filled markers for untouched keypoints and hollow rings for edited ones. -->

A timeline widget docked at the bottom of the viewer embeds a `matplotlib` canvas that plots one bar per frame. Instead of stepping through the video frame by frame, the user can scan the whole recording at a glance and click a bar to jump straight to a frame they have already edited.

The canvas is zoomable and scrollable: the user can zoom in and pan along the time axis to work through a dense region frame by frame, then double-click to snap back to the global view of the whole recording. This keeps the timeline usable for long recordings with thousands of frames.

<!-- Screenshot: the timeline dock widget below the viewer, with blue bars marking edited frames. -->

For multi-animal datasets, the timeline can be split into one row per individual, so the user can see not just which frames were edited but which animal each edit belongs to.

<!-- Screenshot: display individuals
 -->


### Keeping the layers in sync

Because the `Points` and `Tracks` layers are two views of the same data, an edit to one has to propagate to the other. When a point is dragged or deleted, the corresponding vertex of the trajectory is updated in place, so the track line follows the correction immediately instead of only after the file is reloaded.

<!-- Screenshot: tracks being updated from a Point edit. Show both drag and delete. 
 -->

### Saving

The Save widget ties it together: it calls `napari_layers_to_ds()` on the edited layer and writes the result to a NetCDF file through `movement.io`, so the corrected tracks come out in the same format they went in. Before saving, it checks that the `Points` layer was actually created by `movement`'s loader, so trying to save an arbitrary napari layer as a pose dataset fails cleanly.

<!-- Screenshot: the Save panel. -->

### Reloading an edited dataset

Curating a long recording is rarely a single sitting, so the edits have to survive being closed and reopened. 

The file the Save widget writes is a `movement` dataset that also carries the `edited` property. Loading the `.nc` back into the GUI restures the full editing state: previously edited keypoints shown as rings, the timeline re-populates its bars, and any further correction accumulates on top of the earlier ones. This way, a session can be picked up exactly where it was left. 

<!-- Screenshot: a reloaded dataset, with rings and timeline bars from a previous session already in place. -->


## What I did

This project grew out of my own work with pose-estimation data. I was analysing drone videos of seabirds and using `movement` to study their trajectories when I ran into a large number of identity swaps. Correcting them by hand took a long time and was tedious with the tools I had. It struck me that being able to manually curate tracks would help not only me but anyone working with multi-animal pose-estimation data, and that is what led me to apply to GSoC to build it.

I had already written a prototype before the program, without knowing the `movement` codebase well. During GSoC my mentors guided me through turning it into something maintainable: how to plan the work, and how to design the core carefully so that later features would not run into structural problems. We met weekly to discuss progress, and I also had the chance to meet the team in person at FENS 2026 in Barcelona.

The outcome of the coding period is an edit-and-save widget that is about to be released. Manual identity-swap correction is still in progress and will be part of my continued collaboration with `movement` after GSoC.

Partway through the program I gave a short talk about the tool at the Neuroinformatics Unit's summer school. Preparing it pushed me to think about how to organise and structure the information a user needs, which fed directly into the documentation. Presenting to researchers who work with pose-estimation data every day also gave me usability feedback straight from its intended users, and several of their ideas have shaped where the tool is heading.

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

### Making one function handle every editing scenario

The hardest part of the project was `napari_layers_to_ds`, the function that rebuilds a `movement` dataset from the edited napari layers. It sits at the centre of everything: nothing a user does in the GUI is real until it can be written back losslessly, so every new editing feature ended up landing on this one function. It started as a simple inverse of the loader, but each editing scenario forced it to grow. A dragged point brings new coordinates, a deleted point leaves no trace in the layer and has to be told apart from a detection that was always missing, an emptied keypoint or individual should disappear from the dataset while an emptied frame should not. Holding all of these cases in one coherent implementation, without one fix breaking another, was the part I spent the most time on.

### Learning to test

Testing that implementation was a challenge of its own. Good tests here mean being deliberately adversarial. Imagining every way a user could push the widget into a strange state and checking the function still produces a valid dataset. I had less experience with that kind of testing coming in, and at the start of the coding period it was genuinely tough: I found it hard to know what a thorough test suite should even cover. It grew on me, though. Writing the tests turned out to be one of the best ways to understand the design. Every awkward case I had to construct a test for was a case the implementation had to account for, and thinking through them often changed the code itself. By the end I was enjoying it.


## Future work

GSoC gave the widget its foundations, but the full tool we designed in [issue #993](https://github.com/neuroinformatics-unit/movement/issues/993) is not finished yet. After the program I plan to keep contributing to `movement` to complete it: closing the remaining subissues tracked under that issue: [issue #996](https://github.com/neuroinformatics-unit/movement/issues/996), [issue #1006](https://github.com/neuroinformatics-unit/movement/issues/1006), [issue #998](https://github.com/neuroinformatics-unit/movement/issues/998), [issue #1009](https://github.com/neuroinformatics-unit/movement/issues/1009), [issue #1052](https://github.com/neuroinformatics-unit/movement/issues/1052), [issue #1059](https://github.com/neuroinformatics-unit/movement/issues/1059), [issue #1060](https://github.com/neuroinformatics-unit/movement/issues/1060)

Beyond that, I want to keep improving usability, respond to issues on the repository as people start using the tool, and keep the GUI components stable across releases. I will also join the fortnightly `movement` community calls to stay up to date with where the package is heading and make sure the widget evolves along with it.

## Conclusions

By the end of the summer, `movement` had gone from having no way to edit tracks to having a working napari widget for it: users can load their pose data over the source video, drag, add and remove keypoints, watch the trajectories update as they go, keep track of every change they have made, and save the corrected dataset straight back to `movement`'s format.

The widget is not finished. Identity-swap correction and a few other pieces from the original design are still ahead. But the foundations are in place, and I intend to keep building on them.

On a personal level, this was my first time taking something from a rough prototype to a feature other people will actually use, and most of what I learned was about the parts around the code rather than the code itself: designing an implementation before writing it, splitting the work into reviewable pieces, responding to review, and testing an idea hard enough to trust it. I came in more comfortable with the science than with the software engineering, and I leave a lot more confident in both. I'm grateful to GSoC, to the Neuroinformatics Unit, and to my mentors for making that possible, and I'm looking forward to staying part of the `movement` community.

## Acknowledgements

I want to thank my mentors, [Sofía Miñano](https://github.com/sfmig), [Niko Sirmpilatze](https://github.com/niksirbi), and [Chang Huan Lo](https://github.com/lochhh), for their guidance throughout the project. I learned a lot from them. Especially about writing more efficient code, working as a team in software development, designing the core functions of an implementation, and testing everything thoroughly. These are methods I feel I can carry with me into future work in open-source, and I found the whole process genuinely inspiring. I'm also very grateful for the attention they gave to me and to the project. The regular communication has been fundamental to this work, and I'm thankful for that.

Thanks to [Adam Tyson](https://github.com/adamltyson), head of the Neuroinformatics Unit, for hosting the project and making this opportunity possible. And thank you for sending nice stickers and a `movement` mug! 

I would also like to thank [Juan Nunez-Iglesias](https://gist.github.com/jni) for generously sharing a snippet showing how to embed a Matplotlib canvas inside a napari dock widget, on `napari`'s zulip channel. It was both useful and inspiring for this project.

Finally, thank you to Google and the Google Summer of Code program for the opportunity, and for supporting new contributors as they take their first steps into open-source development.

## Related links

- [`movement` repository](https://github.com/neuroinformatics-unit/movement)
- [`movement` documentation](https://movement.neuroinformatics.dev/)
- [My GitHub profile](https://github.com/anna-teruel)