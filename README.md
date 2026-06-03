# GitHub Contribution Log

## Contribution 1: Optimize trampled data in MemoryPacking

**Contribution Number:** 1 / 2 / 3
**Student:** Jacky Li
**Issue:** [WebAssembly/binaryen Issue #3244](https://github.com/WebAssembly/binaryen/issues/3244)
**Status:** [Phase I / Phase II / Phase III / Phase IV] — [In Progress / Complete]

---

## Why I Chose This Issue

I chose this issue because it connects directly to the kind of low-level systems thinking I've developed through both coursework and hands-on projects. Working with assembly and microcontrollers has given me a strong intuition for how memory is laid out and how compilers decide what actually gets emitted into a binary — and MemoryPacking, a pass that decides which memory segments to emit and how, sits right in that same space.

My work in edge AI deepens that interest further. WebAssembly is increasingly used to deploy models in portable, low-footprint environments, and optimizing models for embedded and low-power devices means I'm constantly thinking about memory size, layout, and what data is genuinely necessary versus wasted overhead. That's exactly the concern at the heart of this issue: data that gets trampled (overwritten by an overlapping segment) is dead weight, and emitting it anyway costs binary size for no benefit. I've also worked with neuromorphic and event-based hardware and researched SNNs and TinyML, so I'm comfortable reasoning about systems where every byte and every redundant operation matters.

What sealed it for me is that this isn't just a bug fix — it's an optimization improvement. The current behavior detects trampled data and bails out of optimizing the segment entirely; the better approach is to recognize that trampled data doesn't need to be emitted in the first place. Reasoning correctly about overlapping segments and what can be safely dropped is the kind of careful, correctness-sensitive optimization work I want to get better at, and I think my background puts me in a good position to think it through carefully.

---

## Understanding the Issue

### Problem Description
[In your own words, what's broken or missing?]

### Expected Behavior
[What should happen?]

### Current Behavior
[What actually happens?]

### Affected Components
[Which parts of the codebase are involved?]

---

## Reproduction Process

### Environment Setup
[Notes on setting up your local development environment — challenges you faced, how you solved them]

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
[Your analysis of the root cause — what's causing the issue?]

### Proposed Solution
[High-level description of your fix approach]

### Implementation Plan (UMPIRE Framework)

- **Understand:** [Restate the problem]
- **Match:** [What similar patterns/solutions exist in the codebase?]
- **Plan:**
  1. [Modify file X to do Y]
  2. [Add function Z]
  3. [Update tests]
- **Implement:** [Link to your branch/commits as you work]
- **Review:** [Self-review checklist — does it follow the project's contribution guidelines?]
- **Evaluate:** [How will you verify it works?]

---

## Testing Strategy

### Unit Tests
- Test case 1: [Description]
- Test case 2: [Description]
- Test case 3: [Description]

### Integration Tests
- Integration scenario 1
- Integration scenario 2

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

**PR Description:**
[Draft or final PR description — much of the content above can be adapted]

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

### Resources Used
- [Link to helpful documentation]
- [Tutorial or Stack Overflow post that helped]
- [GitHub issues or discussions that helped]
