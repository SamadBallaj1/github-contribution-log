# Contribution 1: Add a metaflac backend to the replaygain plugin

**Contribution Number:** 1  
**Student:** Samad Ballaj (@SamadBallaj1)  
**Issue:** [beetbox/beets #1203 - replaygain: metaflac backend](https://github.com/beetbox/beets/issues/1203)  
**Fork:** https://github.com/SamadBallaj1/beets  
**Status:** Phase II In Progress

---

## Why I Chose This Issue

beets is a music library tool I can actually read, and it's Python, which is what I work in. The replaygain plugin already has a few backends (command, gstreamer, audiotools, ffmpeg), so adding a metaflac one means following a pattern that's already in the file instead of inventing something new. That feels like the right size for a first contribution.

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

I cloned my fork next to my course notes and added the upstream `beetbox/beets` as a second remote. My fork was 79 commits behind upstream, so I fast-forwarded `master` to upstream before branching, then made a branch called `fix-issue-1203`.

beets builds with poetry and runs its tasks through poethepoet, which it says in `CONTRIBUTING.rst`. I didn't have either, so I added them with `uv tool install poetry poethepoet`. My machine's default Python is 3.14, which is newer than what beets pins against, and I didn't want to fight missing wheels, so I pointed poetry at Python 3.12.9 with `poetry env use`. After that, `poetry install` built the virtualenv and pulled beets 2.11.0 plus the test dependencies cleanly.

For external tools I have ffmpeg but not the GStreamer Python bindings, mp3gain, or metaflac itself. That's enough to reproduce the issue.

To check the environment was healthy I ran `poetry run pytest test/test_util.py`, which gave 34 passed and 2 skipped (the skips are platform things, like a case-sensitive filesystem check). Then I ran the replaygain tests, `poetry run pytest test/plugins/test_replaygain.py`. Twelve tests collect, all gstreamer, and they skip because I don't have the GStreamer bindings. The ffmpeg and command test classes don't get collected at all. I dug into why: their mixins (`FfmpegBackendMixin`, `CmdBackendMixin`) never implement the `test_backend` method that `BackendMixin` marks abstract, so those classes stay abstract and pytest drops them. I confirmed it with `inspect.isabstract`. That's already true on master, not something I changed, but it's a warning for later: my metaflac test mixin has to implement `test_backend` the way `GstBackendMixin` does, or my new tests won't run either.

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

I ran it twice and got the same error both times. As a sanity check I switched the backend to `ffmpeg`, and that one gets past this check (it then runs its own "is ffmpeg installed" check), which confirmed the failure is specifically that metaflac isn't a registered backend.

### Reproduction Evidence

- **Commit showing reproduction:** this issue is a missing feature, not a code bug, so there's nothing to commit as a reproduction. The proof is the config above and the error it produces. My working branch is [fix-issue-1203](https://github.com/SamadBallaj1/beets/tree/fix-issue-1203).
- **Screenshots/logs:** the `UserError` text above is the output that matters.
- **My findings:** the plugin checks the configured backend name against a fixed registry and rejects anything that isn't in it. metaflac just isn't in the list. There's also a [related issue, #1549](https://github.com/beetbox/beets/issues/1549), asking for the same feature, so people still want it.

---

## Solution Approach

### Analysis

[Your analysis of the root cause - what's causing the issue?]

### Proposed Solution

[High-level description of your fix approach]

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** [Restate the problem]

**Match:** [What similar patterns/solutions exist in the codebase?]

**Plan:** [Step-by-step implementation plan]
1. [Modify file X to do Y]
2. [Add function Z]
3. [Update tests]

**Implement:** [Link to your branch/commits as you work]

**Review:** [Self-review checklist - does it follow the project's contribution guidelines?]

**Evaluate:** [How will you verify it works?]

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
