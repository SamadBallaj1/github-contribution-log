# Contribution 1: Add a metaflac backend to the replaygain plugin

**Contribution Number:** 1  
**Student:** Samad Ballaj (@SamadBallaj1)  
**Issue:** [beetbox/beets #1203 - replaygain: metaflac backend](https://github.com/beetbox/beets/issues/1203)  
**Fork:** https://github.com/SamadBallaj1/beets  
**Status:** Phase II In Progress

---

## Why I Chose This Issue

beets is a music library tool I can actually read, and it's Python, which is what I work in. The replaygain plugin already has a few backends (command, gstreamer, audiotools, ffmpeg), so adding a metaflac one means following a pattern that's already in the file instead of inventing something new. Good size for a first PR.

I also want to learn how a real plugin system is structured. The maintainer left a note on the issue about the right approach, so I have somewhere to start and someone to check my plan against.

---

## Understanding the Issue

### Problem Description

The replaygain plugin computes ReplayGain values through a backend. Right now there's no metaflac backend, so people who only have metaflac available (for example on a NAS where the other tools won't install) can't use the plugin for their FLAC files.

### Expected Behavior

You can set `replaygain.backend: metaflac` and beets uses metaflac for FLAC files. Concretely, done looks like:
- a metaflac backend that's selectable in config
- it reads or writes ReplayGain tags on FLAC through metaflac
- it fails with a clear message if metaflac or the needed flag isn't there
- a test covers it, matching the existing backend tests

### Current Behavior

There is no metaflac option. The plugin supports command, gstreamer, audiotools, and ffmpeg only.

### Affected Components

- `beetsplug/replaygain.py` - the plugin and its backend classes. A new backend would follow the existing backend structure there.
- The plugin's tests.

The maintainer [suggested](https://github.com/beetbox/beets/issues/1203#issuecomment-68812475) updating the plugin now that the backends are extensible, and failing gracefully if the flag isn't there. There's also a [related discussion](https://github.com/beetbox/beets/discussions/4935) on how the backends behave.

---

## Reproduction Process

### Environment Setup

- Cloned my fork next to my notes, added `beetbox/beets` as `upstream`. Fork was 79 commits behind, so I pulled `master` current before branching `fix-issue-1203`.
- beets uses poetry and poe (it's in `CONTRIBUTING.rst`). Didn't have them, so `uv tool install poetry poethepoet`.
- My default Python is 3.14, too new for clean wheels, so I pinned poetry to 3.12.9 with `poetry env use`. Then `poetry install` pulled beets 2.11.0 and the test deps.
- Only have ffmpeg locally, not gstreamer, mp3gain, or metaflac. Enough to reproduce this.
- Baseline: `poetry run pytest test/test_util.py` -> `34 passed, 2 skipped`.
- `poetry run pytest test/plugins/test_replaygain.py` -> only 12 collect, all gstreamer, all skipped (no bindings). The ffmpeg and command backend tests don't run on master either, since their mixins skip `test_backend`. So my metaflac test has to implement `test_backend` to actually run.

### Steps to Reproduce

1. Set up the dev environment as above, on the `fix-issue-1203` branch.
2. Point beets at a throwaway config directory and put this in its `config.yaml`:

   ```yaml
   plugins: replaygain
   replaygain:
     backend: metaflac
   ```

3. Run any beets command that loads plugins, for example `beet ls`.

**Expected:** beets accepts the `metaflac` backend and uses it for FLAC files.

**Actual:** the plugin fails to load and beets raises:

   ```
   beets.exceptions.UserError: Selected ReplayGain backend metaflac is not supported. Please select one of: command, gstreamer, audiotools, ffmpeg
   ```

Ran it twice, same error both times. As a sanity check I switched the backend to `ffmpeg`. That one gets past this check (then it runs its own "is ffmpeg installed" check). So the failure is specifically that metaflac isn't a registered backend.

### Reproduction Evidence

- **Commit showing reproduction:** this issue is a missing feature, not a code bug, so there's nothing to commit as a reproduction. The proof is the config above and the error it produces. My working branch is [fix-issue-1203](https://github.com/SamadBallaj1/beets/tree/fix-issue-1203).
- **Screenshots/logs:** the `UserError` text above is the output that matters.
- **My findings:** the plugin checks the configured backend name against a fixed registry and rejects anything that isn't in it. metaflac just isn't in the list. There's also a related, now-closed request, [#1549](https://github.com/beetbox/beets/issues/1549), for the same feature.

---

## Solution Approach

### Analysis

The plugin keeps its backends in a fixed registry in `beetsplug/replaygain.py`. `BACKEND_CLASSES` is a list of the four backend classes (`CommandBackend`, `GStreamerBackend`, `AudioToolsBackend`, `FfmpegBackend`), and right below it `BACKENDS` turns that list into a dict keyed by each backend's `NAME`. Then `ReplayGainPlugin.__init__` looks up the configured backend name in `BACKENDS`, and if it's not a key there it raises the `UserError` I saw. So it's not a bug in the existing code. There's just no `MetaflacBackend` class and no `metaflac` entry in that registry.

### Proposed Solution

Add a `MetaflacBackend` class that subclasses `Backend` with `NAME = "metaflac"`, and add it to the `BACKEND_CLASSES` list so it registers. I'll model it on `CommandBackend`, since both just drive an external command-line tool. The one wrinkle is how metaflac computes gain. Older metaflac could only write ReplayGain tags straight to the source files, with no analysis-only mode, and that's what earlier contributors got stuck on. Current metaflac has a `--scan-replay-gain` option that analyzes without writing (I confirmed this in the xiph FLAC docs). So my backend can run `metaflac --scan-replay-gain`, parse the gains and peaks out of the output, and hand them back for beets to write, which matches how the other backends behave.

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** beets can't compute ReplayGain for FLAC using metaflac. People who only have metaflac available (on a NAS, for example) can't use the plugin, and selecting `backend: metaflac` errors out instead of working.

**Match:** the `Backend` abstract base class defines the interface every backend implements: `compute_track_gain` and `compute_album_gain`. `CommandBackend` is the closest existing example, since it shells out to a tool, checks the tool is installed, and parses its output. The tests use a per-backend mixin pattern, and I found out the hard way that the mixin has to implement `test_backend` or the test class won't even get collected.

**Plan:**
1. Add `MetaflacBackend(Backend)` to `beetsplug/replaygain.py` with `NAME = "metaflac"`. Check metaflac is on the PATH and fail with a clear message if it isn't.
2. Implement `compute_track_gain` and `compute_album_gain` by calling `metaflac --scan-replay-gain` and parsing the track and album gain and peak values.
3. Register the class in `BACKEND_CLASSES`.
4. Add a `MetaflacBackendMixin` and a CLI test to `test/plugins/test_replaygain.py`, following the existing backend tests, implementing `test_backend` so it actually runs, with a FLAC fixture.
5. Document the new backend in `docs/plugins/replaygain.rst` and add a changelog entry to `docs/changelog.rst`.

**Implement:** I haven't written the code yet. It'll go on the [fix-issue-1203](https://github.com/SamadBallaj1/beets/tree/fix-issue-1203) branch.

**Review:** run `poe format` and `poe lint` (ruff), keep the diff scoped, and follow `CONTRIBUTING.rst` (f-strings, the logging shim, and code + tests + docs + changelog).

**Evaluate:** the new metaflac test passes and the existing replaygain tests still pass. Then install metaflac and run `beet replaygain` on real FLAC to confirm tags get written. One thing to watch: metaflac wants all the files at the same sample rate and channels, mono or stereo.

---

## Testing Strategy

### Unit Tests

- [ ] Test case 1: [Description]
- [ ] Test case 2: [Description]
- [ ] Test case 3: [Description]

### Integration Tests

- [ ] Integration scenario 1
- [ ] Integration scenario 2

### Manual Testing

[What you tested manually and results]

---

## Implementation Notes

### Week [X] Progress

[What you built this week, challenges faced, decisions made]

### Week [Y] Progress

[Continue documenting as you work]

### Code Changes

- **Files modified:** [List]
- **Key commits:** [Links to important commits]
- **Approach decisions:** [Why you chose certain approaches]

---

## Pull Request

**PR Link:** [GitHub PR URL when submitted]

**PR Description:** [Draft or final PR description - much of the content above can be adapted]

**Maintainer Feedback:**
- [Date]: [Summary of feedback received]
- [Date]: [How you addressed it]

**Status:** [Awaiting review / Iterating / Approved / Merged]

---

## Learnings & Reflections

### Technical Skills Gained

[What you learned technically]

### Challenges Overcome

[What was hard and how you solved it]

### What I'd Do Differently Next Time

[Reflection on your process]

---

## Resources Used

- [Link to helpful documentation]
- [Tutorial or Stack Overflow post that helped]
- [GitHub issues or discussions that helped]
