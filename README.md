# Contribution 1: Add a metaflac backend to the replaygain plugin

**Contribution Number:** 1  
**Student:** Samad Ballaj (@SamadBallaj1)  
**Issue:** [beetbox/beets #1203 - replaygain: metaflac backend](https://github.com/beetbox/beets/issues/1203)  
**Fork:** https://github.com/SamadBallaj1/beets  
**Status:** Phase IV Complete (PR merged)

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

- Forked beets, added `beetbox/beets` as upstream, made a branch `fix-issue-1203`.
- beets uses poetry (it's in `CONTRIBUTING.rst`). Installed it with `uv tool install poetry poethepoet`.
- My Python 3.14 was too new, so I used 3.12.9, then ran `poetry install`.
- I have ffmpeg but not metaflac yet. Enough to reproduce this.

### Steps to Reproduce

1. In a beets config, turn on the replaygain plugin and set the backend to metaflac:

   ```yaml
   plugins: replaygain
   replaygain:
     backend: metaflac
   ```

2. Run `beet ls`.

**Expected:** beets uses metaflac for FLAC files.

**Actual:** it errors out:

   ```
   UserError: Selected ReplayGain backend metaflac is not supported. Please select one of: command, gstreamer, audiotools, ffmpeg
   ```

Got the same error every time.

### Reproduction Evidence

- **Commit showing reproduction:** it's a missing feature, not a code bug, so there's nothing to commit yet. The config and error above are the proof. Working branch: [fix-issue-1203](https://github.com/SamadBallaj1/beets/tree/fix-issue-1203).
- **Screenshots/logs:** the `UserError` output above is the log. It's a command-line error, so there's no UI to screenshot.
- **My findings:** the plugin checks the backend name against a fixed list and rejects anything that isn't in it. metaflac just isn't there.

---

## Solution Approach

### Analysis

The backends live in a fixed list, `BACKEND_CLASSES`, in `beetsplug/replaygain.py`, and `ReplayGainPlugin` errors if your backend name isn't in it. There's just no `MetaflacBackend`, so `metaflac` isn't an option. It's not a bug, just a missing backend.

### Proposed Solution

Add a `MetaflacBackend` modeled on the existing `CommandBackend` (both run an external tool), and register it. It'd call `metaflac --scan-replay-gain` to read the values without changing the files, then hand them to beets.

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** no metaflac backend exists, so `backend: metaflac` errors out. People who only have metaflac can't use the plugin on FLAC.

**Match:** copy the pattern from `CommandBackend`. `Backend` sets the two methods to fill in, `compute_track_gain` and `compute_album_gain`. `git log` shows the ffmpeg backend was added the same way back in 2018 (`c3af5b3`), so this is a well-worn path.

**Plan:**
1. Add a `MetaflacBackend` to `beetsplug/replaygain.py` and register it in `BACKEND_CLASSES`.
2. Fill in `compute_track_gain` and `compute_album_gain` using `metaflac --scan-replay-gain`.
3. Add a test in `test/plugins/test_replaygain.py` (its mixin needs `test_backend` or it won't run), and update `docs/plugins/replaygain.rst` and `docs/changelog.rst`.

**Implement:** not written yet, goes on the [fix-issue-1203](https://github.com/SamadBallaj1/beets/tree/fix-issue-1203) branch.

**Review:** run `poe lint`, keep it small, follow `CONTRIBUTING.rst`.

**Evaluate:** the new test and the existing ones pass, and `beet replaygain` works on a real FLAC. One thing to watch: metaflac scanning wants all the files at the same sample rate and channels, and only mono or stereo.

---

## Testing Strategy

### Unit Tests

- [x] `test_metaflac_backend_parses_replaygain_tags` feeds the backend a sample of metaflac's `NAME=VALUE` output and checks it pulls out the gain (`-11.55 dB` becomes `-11.55`) and the peak. It runs even when metaflac isn't installed, so the parsing is always covered.

### Integration Tests

- [x] `TestReplayGainMetaflacCli` reuses the same CLI test class the other backends use, pointed at the `whitenoise.flac` fixture. It runs `beet replaygain` for real through metaflac and checks the track and album gains get written and read back. 7 of these pass; the 4 that skip are the Opus (R128) cases, which metaflac doesn't handle.

### Manual Testing

Before I wrote the parser I ran `metaflac --add-replay-gain` and then `--show-tag` on a copy of the FLAC fixture to see the exact output (`REPLAYGAIN_TRACK_GAIN=-11.55 dB`). Once the code was in, `poe test test/plugins/test_replaygain.py` gave 7 passed, 16 skipped (the skips are gstreamer, ffmpeg, and mp3gain, none of which are installed on my machine). `ruff check`, `ruff format`, and `mypy` all come back clean.

---

## Implementation Notes

### Week 3 Progress

I added a `MetaflacBackend` to `beetsplug/replaygain.py` and registered it in `BACKEND_CLASSES`, so you can now pick it with `backend: metaflac`. It shells out to `metaflac --add-replay-gain` to compute the gain, then reads the values back with `metaflac --show-tag`. I modeled it on the existing `CommandBackend`, since both wrap an external command-line tool.

The backend only handles FLAC and skips anything else. For a single track it scans the file on its own; for an album it hands the whole set to metaflac in one call, which is how metaflac works out the album gain.

### Challenges Faced

Two things tripped me up. First, metaflac writes the ReplayGain tags into the FLAC itself, unlike the `command` and `ffmpeg` backends that just print numbers, so I had to run it and then read the tags back with `--show-tag` instead of parsing one command's output. Second, metaflac always targets an 89 dB reference, so my gains came out slightly off when I tried a non-default `targetlevel`. I fixed that by adding `target_level - 89`, the same adjustment the `command` backend makes, and `test_targetlevel_has_effect` passes now.

### Code Changes

- **Files modified:**
  - `beetsplug/replaygain.py` (the new `MetaflacBackend`, registered in `BACKEND_CLASSES`)
  - `test/plugins/test_replaygain.py` (a `MetaflacBackendMixin`, the `TestReplayGainMetaflacCli` class, and a parsing unit test)
  - `docs/plugins/replaygain.rst` and `docs/changelog.rst` (docs and a changelog line)
- **Key commits:** the full list lives on the PR at https://github.com/beetbox/beets/pull/6800/commits (the backend itself, the docs and changelog, the filter() fix, the CI flac install, and the docs code-block). beets moves fast, so I rebase onto master when it drifts and the individual hashes change each time.
- **Branch:** https://github.com/SamadBallaj1/beets/tree/fix-issue-1203
- **Approach decisions:** I reused the project's existing backend test harness instead of writing a new one. metaflac's known limits (FLAC only, and every file in an album needs the same sample rate and channel layout) I left in place and documented in the plugin docs rather than working around them. I also made the tag reader turn any bad metaflac output into a skip-this-file error instead of letting it crash the whole run, the same way the other backends behave.

---

## Pull Request

**PR Link:** https://github.com/beetbox/beets/pull/6800

**PR Description:** Adds a metaflac backend to the replaygain plugin (issue #1203). It runs `metaflac --add-replay-gain` to compute the gain and reads it back with `metaflac --show-tag`, modeled on the existing command backend. FLAC only.

**Maintainer Feedback:**
- Jul 5: @snejus (maintainer) reviewed and requested two changes: use filter() in the metaflac track-gain loop, and install flac in ci.yaml so the tests run on CI. Pushed both fixes.
- Jul 5: Confirmed I tested it locally (ran it on a FLAC file, correct ReplayGain values written) and re-requested review.
- Jul 12: Rebased onto the latest master to clear another changelog conflict, since beets shipped more changes while the PR sat open. Tests still green.
- Jul 14: @snejus took another look and asked me to highlight the docs config example and rebase again. I changed the `::` block to a `.. code-block:: yaml` so the yaml renders highlighted, and rebased onto master. All 18 checks green.
- Jul 14: @snejus approved and merged the PR (merge commit `17d6798`). Issue #1203 is now closed.

**Status:** Merged

---

## Learnings & Reflections

### Technical Skills Gained

I learned how beets' replaygain backends are built: you subclass `Backend`, fill in `compute_track_gain` and `compute_album_gain`, and register the class. I also got more comfortable calling a CLI tool from Python and reading its output back, and writing tests against the project's own test harness.

### Challenges Overcome

The part that threw me was that metaflac writes the ReplayGain tags into the file instead of printing them, so I had to run it and then read the tags back. After I rebased, beets had just released 2.12.0, so my changelog line landed under the released section and CI caught it. I moved it under Unreleased.

### What I'd Do Differently Next Time

Rebase onto upstream more often instead of once at the end. My only real conflict was the changelog, and it only happened because beets shipped 2.12.0 while my PR was open, so my line ended up in a section that had already been released. The beets PR template actually says to add the changelog entry only once review is nearly done, since that file conflicts more than any other, and I get why now. Next time I'll keep that entry under Unreleased and add it late, and treat any busy shared file as the last thing I touch, not the first. The bigger takeaway for me: on an active project the base branch keeps moving while you work, so small frequent rebases beat one big catch-up at the end.

---

## Resources Used

- beets `CONTRIBUTING.rst` and the existing backends in `beetsplug/replaygain.py`
- metaflac docs: https://xiph.org/flac/documentation_tools_metaflac.html
- The issue and discussion: https://github.com/beetbox/beets/issues/1203
