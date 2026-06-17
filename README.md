# github-contribution-log

# Contribution [1]: Inconsistent reference-file naming — drop leftover global numeric prefixes

**Contribution Number:** 1
**Student:** Parker Cassar
**Issue:** https://github.com/sebastian-software/skills/issues/10
**Status:** Phase I — Issue Selection

---

## Why I Chose This Issue

I chose this issue because it has a clear, well-defined scope: rename leftover numeric-prefixed reference files to descriptive kebab-case and update the corresponding links in each `SKILL.md`. The maintainer has already documented the problem, listed affected files, and described the validation step (`pnpm skill-sync validate`), which makes it a strong fit for getting started in a new repository.

It is labeled `good first issue` and `enhancement`, so the work is mechanical rather than architectural — a good way to learn the repo's skill structure and contribution workflow without needing deep domain knowledge upfront. I also commented on the issue to claim it and am waiting for assignment.

---

## Understanding the Issue

### Problem Description

When the original monolithic skill set was split into separate skills, most reference files were renamed to clean, descriptive kebab-case (e.g. `component-api-design.md`). Fifteen internal skills still carry global numeric prefixes from the old monolith (ranging from `01` to `36`). Those numbers no longer mean anything now that each skill lives in its own directory.

### Expected Behavior

Reference files should use descriptive kebab-case names only — no leading numeric prefixes. For example, `23-html-accessibility.md` should become `html-accessibility.md`.

### Current Behavior

Fifteen skills still have prefixed reference files under their `references/` directories. Examples:

- `s7n-accessibility-html/references/23-html-accessibility.md`
- `s7n-auth-security-ux/references/34-auth-web-security.md`
- `s7n-forms-ux/references/24-forms-ux.md`
- `s7n-css-architecture/references/33-baseline-support.md`, `36-css-build-tooling.md`
- `s7n-layout-spacing/references/26-responsive-design.md`, `27-css-layout-responsive.md`

Newer skills already follow the target convention, e.g. `s7n-react-component/references/component-api-design.md`.

### Affected Components

- 15 skill directories with `references/` files that still use numeric prefixes
- Corresponding `SKILL.md` files that link to those reference paths

### Suggested Fix (from issue)

1. Rename each numeric-prefixed file to its kebab-case equivalent (drop the leading number and hyphen).
2. Update every matching link in the affected `SKILL.md` files.
3. Run `pnpm skill-sync validate` to confirm no dangling reference links remain.

---

## Reproduction Process

### Environment Setup

[Not started — Phase II]

### Steps to Reproduce

[Not started — Phase II]

### Reproduction Evidence

- **Commit showing reproduction:** [Link to commit in your fork]
- **Screenshots/logs:** [If applicable]
- **My findings:** [What you discovered during reproduction]

---

## Solution Approach

### Analysis

[Not started — Phase II]

### Proposed Solution

[Not started — Phase II]

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** [Restate the problem]

**Match:** [What similar patterns/solutions exist in the codebase?]

**Plan:** [Step-by-step implementation plan]
1. [Modify file X to do Y]
2. [Add function Z]
3. [Update tests]

**Implement:** [Link to your branch/commits as you work]

**Review:** [Self-review checklist — does it follow the project's contribution guidelines?]

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

### Week 1 Progress (Phase I)

- Selected issue [#10](https://github.com/sebastian-software/skills/issues/10) in `sebastian-software/skills`
- Commented on the issue to request assignment
- Created this Contribution README

### Week [X] Progress

[Continue documenting as you work]

### Code Changes

- **Files modified:** [List]
- **Key commits:** [Links to important commits]
- **Approach decisions:** [Why you chose certain approaches]

---

## Pull Request

**PR Link:** [GitHub PR URL when submitted]

**PR Description:** [Draft or final PR description — much of the content above can be adapted]

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

- [Issue #10 — Inconsistent reference-file naming](https://github.com/sebastian-software/skills/issues/10)
- [sebastian-software/skills repository](https://github.com/sebastian-software/skills)
