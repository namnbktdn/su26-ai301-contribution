# Contribution 1: Defining the data size of chunks

**Contribution Number:** 1

**Student:** Nam Hoang

**Issue:** https://github.com/zarr-developers/zarr-python/issues/270

**Status:** Phase I

---

## Why I Chose This Issue

Zarr-Python is actually one of the tools I’m currently using in my research, where I’m integrating a new data format for biomedical imaging data in my field. Because of that, understanding the Zarr library more deeply would directly support my research, and this issue seems like a great first step toward learning the codebase.

---

## Understanding the Issue

### Problem Description

This issue acts as a good-first enhancement issue. It proposed usng a the "disk size" units to define the chunk, instead of dimensions. 

### Expected Behavior

For example, zarr.zeros(..., chunks="100MB", ...) will chunk in a way that each chunk is 100MB. 

### Current Behavior

Now, we can only provide the dimension shape for the chunk, like zarr.zeros(..., chunks=(1, 100, 100), ...) and the chunk size will be calculated based on the data shape and the chunk shape. 

### Affected Components



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
