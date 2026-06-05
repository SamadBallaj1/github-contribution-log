# Contribution 1: Add a metaflac backend to the replaygain plugin

**Contribution Number:** 1  
**Student:** Samad Ballaj (@SamadBallaj1)  
**Issue:** [beetbox/beets #1203 - replaygain: metaflac backend](https://github.com/beetbox/beets/issues/1203)  
**Fork:** https://github.com/SamadBallaj1/beets  
**Status:** Phase I In Progress

---

## Why I Chose This Issue

beets is a music library tool I can actually read, and it's Python, which is what I work in. The replaygain plugin already has a few backends (bs1770gain, ffmpeg, command, gstreamer), so adding a metaflac one means following a pattern that's already in the file instead of inventing something new. That feels like the right size for a first contribution.

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

There is no metaflac option. The plugin supports bs1770gain, ffmpeg, command, and gstreamer only.

### Affected Components

- `beetsplug/replaygain.py` - the plugin and its backend classes. A new backend would follow the existing backend structure there.
- The plugin's tests.

The maintainer [suggested](https://github.com/beetbox/beets/issues/1203#issuecomment-68812475) updating the plugin now that the backends are extensible, and failing gracefully if the flag isn't there. There's also a [related discussion](https://github.com/beetbox/beets/discussions/4935) on how the backends behave.

---

## Reproduction Process

### Environment Setup

[Notes on setting up your local development environment - challenges you faced, how you solved them]

### Steps to Reproduce

1. [Step 1]
2. [Step 2]
3. [Observed result]

### Reproduction Evidence

- **Commit showing reproduction:** [Link to commit in your fork]
- **Screenshots/logs:** [If applicable]
- **My findings:** [What you discovered during reproduction]

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
