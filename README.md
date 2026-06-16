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

- Forked beets, added `beetbox/beets` as upstream, made a branch `fix-issue-1203`.
- beets uses poetry (it's in `CONTRIBUTING.rst`). Installed it with `uv tool install poetry poethepoet`.
- My Python 3.14 was too new, so I used 3.12.9, then ran `poetry install`.
- I have ffmpeg but not metaflac yet. Enough to reproduce this.

### Steps to Reproduce

1. In a beets config, turn on the replaygain plugin and set `backend: metaflac`.
2. Run `beet ls`.

**Expected:** beets uses metaflac for FLAC files.

**Actual:** it errors out:

   ```
   UserError: Selected ReplayGain backend metaflac is not supported. Please select one of: command, gstreamer, audiotools, ffmpeg
   ```

Got the same error every time.

### Reproduction Evidence

- **Branch:** [fix-issue-1203](https://github.com/SamadBallaj1/beets/tree/fix-issue-1203)
- It's a missing feature, so there's no code to commit yet. The config and error above are the proof.

---

## Solution Approach

### Analysis

The backends live in a fixed list, `BACKEND_CLASSES`, in `beetsplug/replaygain.py`, and `ReplayGainPlugin` errors if your backend name isn't in it. There's just no `MetaflacBackend`, so `metaflac` isn't an option. It's not a bug, just a missing backend.

### Proposed Solution

Add a `MetaflacBackend` modeled on the existing `CommandBackend` (both run an external tool), and register it. It'd call `metaflac --scan-replay-gain` to read the values without changing the files, then hand them to beets.

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** no metaflac backend exists, so `backend: metaflac` errors out. People who only have metaflac can't use the plugin on FLAC.

**Match:** copy the pattern from `CommandBackend`. `Backend` sets the two methods to fill in, `compute_track_gain` and `compute_album_gain`.

**Plan:**
1. Add a `MetaflacBackend` and register it in `BACKEND_CLASSES`.
2. Fill in the two methods using `metaflac --scan-replay-gain`.
3. Add a test (its mixin needs `test_backend` or it won't run), and a docs + changelog entry.

**Implement:** not written yet, goes on the [fix-issue-1203](https://github.com/SamadBallaj1/beets/tree/fix-issue-1203) branch.

**Review:** run `poe lint`, keep it small, follow `CONTRIBUTING.rst`.

**Evaluate:** the new test and the existing ones pass, and `beet replaygain` works on a real FLAC.

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
