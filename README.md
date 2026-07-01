# github-contribution-log

# Contribution [1]: PyArrow fails to cast UUID FIXED_LEN_BYTE_ARRAY back to `uuid.UUID` on Python 3.14

**Contribution Number:** 1
**Student:** Parker Cassar
**Issue:** https://github.com/apache/arrow/issues/50312
**Fork:** https://github.com/parker-cassar/arrow
**Status:** Phase I — Issue Selection

---

## Why I Chose This Issue

I chose this issue because it's a self-contained regression with a precise, minimal reproduction already provided by the reporter (write a `uuid.UUID` to Parquet, read it back, check the Python type). That makes it a strong first issue: I don't have to hunt for a repro case, I just need to understand why the read-back cast behaves differently across Python versions.

It also touches an area — PyArrow's Python/C++ boundary for extension-type casting — that I haven't worked in before, so it's a good opportunity to learn how PyArrow bridges Arrow's columnar types back to native Python objects. I commented on the issue committing to attempt a fix within 24 hours, so I want to move quickly through reproduction and planning.

---

## Understanding the Issue

### Problem Description

PyArrow stores `uuid.UUID` values written via `pa.Table.from_pandas` as a `FIXED_LEN_BYTE_ARRAY` (16 bytes) in Parquet. On read-back, PyArrow is expected to cast that fixed-length byte array back into `uuid.UUID` objects. On Python 3.14 / nightly builds, that cast is not happening — the reader instead returns raw `bytes`.

### Expected Behavior

Reading a Parquet file written from a `uuid.UUID` column should yield `uuid.UUID` objects in the resulting pandas DataFrame, regardless of Python version — matching the behavior on Python 3.13 and earlier.

### Current Behavior

On Python 3.14 / nightly builds, `type(result_df.loc[0, "id"])` returns `bytes` instead of `uuid.UUID`. On Python 3.13 and below, the same code correctly returns `uuid.UUID`. This is a regression specific to newer Python builds, not a general PyArrow bug.

### Reproduction Snippet (from issue)

```python
import pyarrow as pa
import pyarrow.parquet as pq
import pandas as pd
import uuid

original_uuid = uuid.uuid4()
df = pd.DataFrame({"id": [original_uuid]})
table = pa.Table.from_pandas(df)

pq.write_table(table, "test_uuid.parquet")
read_table = pq.read_table("test_uuid.parquet")
result_df = read_table.to_pandas()

# Python 3.13: uuid.UUID. Python 3.14/nightly: bytes.
print(type(result_df.loc[0, "id"]))
```

### Affected Components

- **Python** — the `to_pandas()` / pandas conversion path that casts Arrow extension/fixed-length-byte-array types back to Python objects
- **Parquet** — the read path that reconstructs typed columns from `FIXED_LEN_BYTE_ARRAY` data

This was surfaced while adding upstream UUID Parquet tests to the pandas test suite (`pandas-dev/pandas#65647`), suggesting the regression is likely tied to a Python 3.14–specific change (e.g. in `uuid`, `pickle`, or C-API internals PyArrow relies on for the cast).

### Suggested Fix

Not yet proposed by the maintainers — this is an open investigation. My plan is to bisect between Python 3.13 and 3.14 behavior to find what changed (likely in how PyArrow's canonical extension type / pandas metadata round-trips, or a Python 3.14 C-API change affecting the cast), then narrow to the specific PyArrow code path responsible.

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

- Forked `apache/arrow` to [parker-cassar/arrow](https://github.com/parker-cassar/arrow)
- Selected issue: PyArrow fails to cast UUID `FIXED_LEN_BYTE_ARRAY` back to `uuid.UUID` on Python 3.14 / nightly
- Commented on the issue committing to attempt a fix within 24 hours
- Updated this Contribution README

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

- [apache/arrow issue — PyArrow UUID Parquet cast regression on Python 3.14] (link TBD)
- [pandas-dev/pandas#65647 — upstream UUID Parquet tests](https://github.com/pandas-dev/pandas/pull/65647)
- [My fork: parker-cassar/arrow](https://github.com/parker-cassar/arrow)
