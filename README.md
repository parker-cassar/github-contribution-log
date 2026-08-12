# github-contribution-log

# Contribution [1]: PyArrow fails to cast UUID FIXED_LEN_BYTE_ARRAY back to `uuid.UUID` on Python 3.14

**Contribution Number:** 1
**Student:** Parker Cassar
**Issue:** https://github.com/apache/arrow/issues/50312
**Fork:** https://github.com/parker-cassar/arrow
**Status:** Phase IV - Submit and Iterate

---

## Why I Chose This Issue

I chose this issue because it's a self-contained regression with a precise, minimal reproduction already provided by the reporter (write a `uuid.UUID` to Parquet, read it back, check the Python type). That makes it a strong first issue: I don't have to hunt for a repro case, I just need to understand why the read-back cast behaves differently across Python versions.

It also touches an area - PyArrow's Python/C++ boundary for extension-type casting - that I haven't worked in before, so it's a good opportunity to learn how PyArrow bridges Arrow's columnar types back to native Python objects. I commented on the issue committing to attempt a fix within 24 hours, so I want to move quickly through reproduction and planning.

---

## Understanding the Issue

### Problem Description

PyArrow stores `uuid.UUID` values written via `pa.Table.from_pandas` as a `FIXED_LEN_BYTE_ARRAY` (16 bytes) in Parquet. On read-back, PyArrow is expected to cast that fixed-length byte array back into `uuid.UUID` objects. On Python 3.14 / nightly builds, that cast is not happening - the reader instead returns raw `bytes`.

### Expected Behavior

Reading a Parquet file written from a `uuid.UUID` column should yield `uuid.UUID` objects in the resulting pandas DataFrame, regardless of Python version - matching the behavior on Python 3.13 and earlier.

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

- **Python** - the `to_pandas()` / pandas conversion path that casts Arrow extension/fixed-length-byte-array types back to Python objects
- **Parquet** - the read path that reconstructs typed columns from `FIXED_LEN_BYTE_ARRAY` data

This was surfaced while adding upstream UUID Parquet tests to the pandas test suite (`pandas-dev/pandas#65647`), suggesting the regression is likely tied to a Python 3.14-specific change (e.g. in `uuid`, `pickle`, or C-API internals PyArrow relies on for the cast).

### Suggested Fix

Not yet proposed by the maintainers - this is an open investigation. My plan is to bisect between Python 3.13 and 3.14 behavior to find what changed (likely in how PyArrow's canonical extension type / pandas metadata round-trips, or a Python 3.14 C-API change affecting the cast), then narrow to the specific PyArrow code path responsible.

---

## Reproduction Process

### Environment Setup

**Branch:** [`gh-50312-uuid-pandas-roundtrip`](https://github.com/parker-cassar/arrow/tree/gh-50312-uuid-pandas-roundtrip) in my fork (named after issue #50312).

**Setup approach:** Followed upstream docs rather than a dev container. Read [CONTRIBUTING.md](https://github.com/apache/arrow/blob/main/CONTRIBUTING.md) for the issue/PR workflow, then the [Python development guide](https://arrow.apache.org/docs/developers/python/index.html) for the CMake + editable-install build. Inspected [`.github/workflows/python.yml`](https://github.com/apache/arrow/blob/main/.github/workflows/python.yml) to see how CI runs Python tests (`pytest python/pyarrow/tests/`) across matrix Python versions - I'll mirror that locally before opening a PR.

**Steps taken:**

1. Forked `apache/arrow` to [parker-cassar/arrow](https://github.com/parker-cassar/arrow)
2. Cloned locally: `git clone https://github.com/parker-cassar/arrow.git && cd arrow`
3. Created working branch: `git checkout -b gh-50312-uuid-pandas-roundtrip`
4. Created isolated venvs for reproduction (PyPI wheels, faster than a full source build for Phase II):
 - Python 3.14.6: `python3.14 -m venv .venv-314 && source .venv-314/bin/activate && pip install pyarrow pandas`
 - Python 3.13.7: same pattern with `python3.13` (control)
 - Python 3.10.13 system install (baseline, already had packages)
5. Platform: macOS (darwin 24.6.0), Homebrew-managed Python builds

**Challenges encountered:**

| Challenge | How I resolved it |
|-----------|-------------------|
| `pip install` on Homebrew Python 3.14 failed with **PEP 668 externally-managed-environment** | Created a venv first (`python3.14 -m venv .venv-314`) instead of installing system-wide |
| **No `pyarrow` module** on a fresh Python 3.14 install | Installed into the venv via `pip install pyarrow pandas` |
| Issue says 3.13 returns `uuid.UUID` but my 3.13 venv also returned `bytes` with PyArrow 24.0.0 | Ran intermediate-type checks (`to_pylist()` vs `to_pandas()`) to isolate the failure to the pandas bridge, not Parquet I/O - see findings below |
| Full Arrow source build is heavy (CMake, C++ toolchain) | Used PyPI wheels for Phase II reproduction; source build deferred to Phase III for writing the fix |

For Phase III I'll build PyArrow from source off `main` on the working branch so I can add a regression test and iterate on a fix without waiting on release wheels.

### Steps to Reproduce

1. Create and activate a Python 3.14 virtual environment:
   ```bash
   python3.14 -m venv .venv-314 && source .venv-314/bin/activate
   pip install pyarrow pandas
   ```
2. Save the following as `repro_uuid.py`:
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

   print(type(result_df.loc[0, "id"]))
   ```
3. Run it: `python repro_uuid.py`
4. **Expected:** `<class 'uuid.UUID'>` - the value written should roundtrip as a Python UUID object.
5. **Actual (Python 3.14):** `<class 'bytes'>` - raw 16-byte blob instead of a UUID.
6. To confirm the failure is in the pandas bridge (not Parquet I/O), add these checks:
   ```python
   print(read_table.column("id").type)                    # extension<arrow.uuid>
   print(type(read_table.column("id").to_pylist()[0]))    # uuid.UUID (works)
   print(type(read_table.to_pandas().loc[0, "id"]))        # bytes (broken)
   ```
7. Repeat steps 1-6 on Python 3.13 as a control. Parquet schema and `to_pylist()` behave identically; `to_pandas()` returns `bytes` on both with PyArrow 24.0.0.

### Reproduction Evidence

- **Issue:** [apache/arrow#50312](https://github.com/apache/arrow/issues/50312)
- **Fork:** [parker-cassar/arrow](https://github.com/parker-cassar/arrow)
- **My findings:**

| Step | Python 3.13 | Python 3.14 |
|------|-------------|-------------|
| `from_pandas` column type | `extension<arrow.uuid>` | `extension<arrow.uuid>` |
| Parquet read column type | `extension<arrow.uuid>` | `extension<arrow.uuid>` |
| `to_pylist()[0]` type | `uuid.UUID` | `uuid.UUID` |
| `to_pandas().loc[0, "id"]` type | `bytes` | `bytes` |

**Key observation:** Parquet write/read preserves the `arrow.uuid` extension type correctly on both Python versions. The Arrow-native path (`to_pylist()`, `UuidScalar.as_py()`) also returns `uuid.UUID` on both. The failure is isolated to the **`to_pandas()` conversion path**, which falls back to returning raw `bytes` from the underlying `fixed_size_binary(16)` storage.

**Root cause (not just symptom):** `UuidType` never implemented `to_pandas_dtype()`, so `_array_like_to_pandas()` in `array.pxi` falls through to the generic C++ `ConvertArrayToPandas` converter, which materializes the underlying `fixed_size_binary(16)` storage as raw `bytes`. The symptom is "bytes instead of UUID on 3.14"; the cause is a missing pandas dtype mapping on a canonical extension type that was added without a corresponding `to_pandas()` path.

**When was this gap introduced?** Used `git log` on the upstream repo:
- `2328b6e` (2024-08-26) - [GH-15058](https://github.com/apache/arrow/pull/37298): native UUID extension type added (`UuidType`, `UuidScalar.as_py()`)
- `75acf37` (2025-04-21) - [GH-43807](https://github.com/apache/arrow/pull/45866): Parquet read/write support for UUID added
- Neither commit added `to_pandas_dtype()` on `UuidType`. The gap has existed since UUID support landed; Python 3.14 / pandas 3.0 likely exposed it because downstream test suites (e.g. `pandas-dev/pandas#65647`) now assert `to_pandas()` roundtrips.

**Relevant code paths identified:**
- `python/pyarrow/types.pxi` - `UuidType`, `UuidScalar`, `UuidArray` definitions
- `python/pyarrow/scalar.pxi` - `UuidScalar.as_py()` correctly returns `UUID(bytes=...)`
- `python/pyarrow/array.pxi` - `_array_like_to_pandas()` extension type branch
- `python/pyarrow/pandas_compat.py` - `_get_extension_dtypes()` for table-level conversion
- `python/pyarrow/public-api.pxi` - registers `arrow.uuid` extension on Parquet read
- `python/pyarrow/tests/parquet/test_data_types.py` - existing `test_uuid_extension_type()` (roundtrip at Arrow level, no `to_pandas` assertion)

---

## Solution Approach

### Analysis

The bug is not in Parquet serialization - UUID data is stored and read back as `extension<arrow.uuid>` with `fixed_size_binary(16)` storage in both Python versions. The regression is in how PyArrow converts that extension array to a pandas Series via `to_pandas()`.

The conversion logic in `_array_like_to_pandas` checks whether an extension type provides a pandas dtype via `to_pandas_dtype()`. If the returned dtype has `__from_arrow__`, it uses that for a typed conversion. `UuidType` has no such mapping, so PyArrow delegates to the generic C++ converter, which materializes the storage bytes rather than calling `UuidScalar.as_py()`.

The fix should ensure that when `to_pandas()` encounters an `extension<arrow.uuid>` column, the result contains `uuid.UUID` objects - matching what `to_pylist()` already produces.

### Proposed Solution (Phase II plan)

Teach the `to_pandas()` path to handle `UuidType` the same way the native Arrow path already does. Two viable approaches at planning time:

1. **Implement `to_pandas_dtype()` on `UuidType`** that returns a pandas-compatible dtype with `__from_arrow__`, converting via `UuidScalar.as_py()`. This follows the pattern used by other extension types (e.g. `ExampleUuidType` in the test suite).
2. **Add a special case in `_array_like_to_pandas`** for `UuidType` that converts through `to_pylist()` or iterates `UuidScalar.as_py()` before building the Series. Simpler but less extensible.

Option 1 was the Phase II preferred plan - it matched existing PyArrow conventions and would also benefit `types_mapper`-based table conversions in `pandas_compat.py`.

### Approach change (documented for Phase IV consistency)

**What changed after review:** Maintainers (@rok) asked to move UUID to Python object conversion into the **C++ pandas conversion path** instead of the Python `to_pandas_dtype()` / `__from_arrow__` compat layer. See [rok/arrow#55](https://github.com/rok/arrow/pull/55) and discussion on [PR #50325](https://github.com/apache/arrow/pull/50325) (2026-07-21).

**Why the change:** cleaner API, better performance, and a neater pandas compat surface than a Python-only dtype wrapper.

**What shipped in the PR:** C++ `UuidFromBytes` helpers + `arrow_to_pandas.cc` integration, regression tests, kwargs/tuple reuse for allocation efficiency, and `to_numpy` keeping storage `bytes`. The Phase II root-cause diagnosis (missing pandas conversion for `UuidType`) still holds; only the implementation layer changed.

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** When reading a Parquet file containing UUIDs, `table.to_pandas()` must return `uuid.UUID` objects, not raw `bytes`. The Arrow layer already handles this correctly; the pandas bridge does not.

**Match:** Phase II matched `MyCustomIntegerType.to_pandas_dtype()` in `test_pandas.py`. After maintainer feedback, the match became @rok's C++ sketch in [rok/arrow#55](https://github.com/rok/arrow/pull/55) and existing UUID helpers in `helpers.cc`.

**Edge cases to handle in the fix:**
- **Null UUIDs** - a nullable UUID column should roundtrip as `None`, not empty bytes
- **Empty tables** - `to_pandas()` on a zero-row UUID column should not crash
- **`types_mapper` override** - if a caller passes a custom mapper, it should take precedence over the default UUID mapping
- **Parquet with `arrow_extensions_enabled=False`** - reading as plain `fixed_size_binary(16)` is expected to stay as bytes; only the extension-typed path needs UUID conversion
- **NumPy conversion** - `to_numpy(zero_copy_only=False)` should return storage `bytes`, not `uuid.UUID`

**Plan (as executed in Phase III/IV):**
1. Open PR against `apache/arrow` `main` with failing/passing regression coverage for UUID to pandas.
2. Integrate C++ conversion path per maintainer proposal.
3. Address review rounds (helpers factoring, kwargs reuse, NumPy storage behavior).
4. Keep CI green and iterate until approvals.

**Implement:** https://github.com/apache/arrow/pull/50325

**Review:** Title prefixed `GH-50312: [Python]`, tests included, review feedback addressed in follow-up commits.

**Evaluate:** Round-trip returns `uuid.UUID` via `to_pandas()`; `to_numpy` returns `bytes`; PR approved by @rok and @AlenkaF (open, awaiting merge).

---

## Testing Strategy

### Unit Tests

- [x] Parquet UUID roundtrip via `to_pandas()` returns `uuid.UUID`
- [x] Null UUID values roundtrip as `None`, not as empty bytes
- [x] UUID `to_numpy(zero_copy_only=False)` returns storage `bytes`

### Integration Tests

- [x] Full issue repro script passes end-to-end (`type(...)` is `uuid.UUID` after fix)
- [x] Existing UUID / extension tests still pass (no regression on touched paths)

### Manual Testing / before-after evidence

```text
# Before
>>> type(result_df.loc[0, "id"])
<class 'bytes'>

# After
>>> type(result_df.loc[0, "id"])
<class 'uuid.UUID'>
```

Also spot-checked `to_pylist()` still returns `uuid.UUID` (unchanged).

---

## Implementation Notes

### Week 1 Progress (Phase I)

- Forked `apache/arrow` to [parker-cassar/arrow](https://github.com/parker-cassar/arrow)
- Selected issue: PyArrow fails to cast UUID `FIXED_LEN_BYTE_ARRAY` back to `uuid.UUID` on Python 3.14 / nightly
- Commented on the issue committing to attempt a fix within 24 hours
- Updated this Contribution README

### Week 2 Progress (Phase II)

- Created working branch `gh-50312-uuid-pandas-roundtrip` in fork
- Read CONTRIBUTING.md and inspected `.github/workflows/python.yml` for CI test patterns
- Cloned fork locally; set up Python 3.13 / 3.14 venvs (resolved PEP 668 blocker)
- Reproduced the issue using the script from [#50312](https://github.com/apache/arrow/issues/50312)
- Confirmed Parquet roundtrip preserves `extension<arrow.uuid>` - the bug is in `to_pandas()`, not Parquet I/O
- Traced the failure to `_array_like_to_pandas()` in `array.pxi` where `UuidType` lacks `to_pandas_dtype()`
- Used `git log` to date the gap: UUID + Parquet support landed in 2024-2025 without a pandas conversion path
- Identified `MyCustomIntegerType.to_pandas_dtype()` in `test_pandas.py` as the pattern to follow
- Drafted fix plan with edge cases (nulls, empty tables, `types_mapper`, `arrow_extensions_enabled=False`)

### Week 8 Progress (Phase IV)

- Posted a status update on [PR #50325](https://github.com/apache/arrow/pull/50325) after integrating @rok's C++ conversion proposal ([rok/arrow#55](https://github.com/rok/arrow/pull/55))
- Comment (2026-07-27): full UUID round-trip was passing; finishing a last pass and would have the update up in the next day or two
- Thanked @rok for the proposal; it made the rewrite much easier than staying on the Python compat-layer approach

### Week 9 Progress (Phase IV)

- @rok reviewed again: "I have a potential performance improvement suggestion, but other than that this looks pretty good."
- Review note on `python/pyarrow/src/arrow/python/helpers.cc`: creating a new kwargs dict for every UUID value was unnecessary; pass a reused empty kwargs into `UuidFromBytes` instead
- Addressed that feedback with two commits:
 - [`ff16379`](https://github.com/apache/arrow/pull/50325/commits/ff163797f2b635d43dd4b5584698cee07c0d5bf9) - `GH-50312: [Python] Reuse empty kwargs`
 - [`2f35759`](https://github.com/apache/arrow/pull/50325/commits/2f35759533ba83986d9d3938b45eb96186929574) - `GH-50312: [Python] Use shared empty tuple constant for UUID construction`

### Week 10 Progress (Phase IV) - Approved, awaiting merge

- @rok: almost there; need `type(uuid_array.to_numpy(zero_copy_only=False)[0])` to be `bytes`, because extension-array NumPy conversion should delegate to storage type (`arrow_to_pandas.cc` + `test_extension_type.py`)
- Bot labeled `awaiting changes`
- Addressed with:
 - [`834be3a`](https://github.com/apache/arrow/pull/50325/commits/834be3a) - Fix lint
 - [`a4f309b`](https://github.com/apache/arrow/pull/50325/commits/a4f309b) - UUID `to_numpy` returns bytes
- Bot moved labels to `awaiting change review`
- Asked @rok to re-run macOS 15-intel CI (Install MinIO / `wget` could not resolve `dl.min.io` before build/tests ran)
- @rok **APPROVED**: "LGTM. I'll wait a bit if @AlenkaF or @pitrou have time to review."
- Bot labeled `awaiting merge`
- @AlenkaF **APPROVED**: thanks for the C++ solution; happy to see it get merged
- Thanked @AlenkaF and @rok; credited the C++ approach to [rok/arrow#55](https://github.com/rok/arrow/pull/55)

### Code Changes

- **Files modified:** `python/pyarrow/src/arrow/python/helpers.cc`, `helpers.h`, `arrow_to_pandas.cc`, `python/pyarrow/tests/parquet/test_data_types.py`, `python/pyarrow/tests/test_extension_type.py`
- **Key commits:**
 - [`f71b3a2`](https://github.com/apache/arrow/pull/50325/commits/f71b3a2) - C++ UUID to pandas conversion (integrated proposal)
 - [`b8c8338`](https://github.com/apache/arrow/pull/50325/commits/b8c8338) - avoid per-element allocations
 - [`352cafd`](https://github.com/apache/arrow/pull/50325/commits/352cafd) - move UUID construction into `helpers.cc` (per @pitrou)
 - [`ff16379`](https://github.com/apache/arrow/pull/50325/commits/ff163797f2b635d43dd4b5584698cee07c0d5bf9) - reuse empty kwargs
 - [`2f35759`](https://github.com/apache/arrow/pull/50325/commits/2f35759533ba83986d9d3938b45eb96186929574) - shared empty tuple constant
 - [`834be3a`](https://github.com/apache/arrow/pull/50325/commits/834be3a) - fix lint
 - [`a4f309b`](https://github.com/apache/arrow/pull/50325/commits/a4f309b) - UUID `to_numpy` returns bytes
- **Approach decisions:** Started from Phase II Python dtype plan; switched to @rok's C++ conversion path after review; then iterated on allocation reuse and NumPy storage semantics.

---

## Pull Request

**PR Link:** https://github.com/apache/arrow/pull/50325

**Summary:** Open (not draft) PR against `apache/arrow` `main` that fixes UUID extension columns returning `bytes` from `to_pandas()` by converting in the C++ pandas path. Uses Arrow's PR template (Rationale / What / Tested / User-facing), includes `Closes #50312`, acceptance checklist, and before/after console evidence.

**PR title:** `GH-50312: [Python] Fix UUID extension type round-trip to pandas returning bytes`

**Current status:** **Approved and awaiting merge** on upstream `main` (bot label `awaiting merge`). Approvals from @rok and @AlenkaF.

**Process / communication:**
- Surfaced to maintainers via repeated @mentions of @rok, @AlenkaF, and @pitrou on the PR conversation
- Reviewers requested in the PR panel (`raulcd`, `jorisvandenbossche`)
- **Course Portal:** submit the check-in form with **Phase IV Complete** marked (student action on portal; not automated here)

### Phase IV deliverables checklist (rubric map)

**Pull Request Submission**
- [x] PR open (not draft) against upstream `apache/arrow` `main`: https://github.com/apache/arrow/pull/50325
- [x] Uses Arrow PR template sections: Rationale / What changes / Are these changes tested? / Are there any user-facing changes?
- [x] Issue linked with close keyword: `Closes #50312`

**PR Description Quality**
- [x] Why before what: **Rationale for this change** appears before **What changes are included in this PR?**
- [x] Acceptance criteria checklist filled in (tests added, tests passing, style, no unintended breaking API)
- [x] Before/after console evidence included (`bytes` vs `uuid.UUID`)

**README Documentation Quality**
- [x] Pull Request section has PR link, summary, and current status
- [x] Maintainer Feedback log has dates, feedback, my response, and commit refs
- [x] Learnings & Reflections filled (technical gains, challenges, what I'd do differently, teachable insight)
- [x] Internal consistency: Phase II planned `to_pandas_dtype()`; Phase IV documents the switch to the C++ path under **Approach change**

**Process & Communication**
- [ ] Course Portal check-in submitted with **Phase IV Complete** marked (do this when sending the README URL)
- [x] Maintainers @mentioned / review requested on the PR (`@rok`, `@AlenkaF`, `@pitrou`; review requests for `raulcd`, `jorisvandenbossche`)

**Stretch / Bonus**
- [x] Full loop: multiple review rounds with substantive follow-up commits; PR **approved** by @rok and @AlenkaF (`awaiting merge`); Learnings includes a teachable insight for future cohorts

**Maintainer Feedback log:**

| Date | Feedback | My response | Commit ref(s) |
|------|----------|-------------|----------------|
| 2026-07-06 | @rok: pandas 2.x/3.x reshape concern; prefer 1-D `__from_arrow__` + reshape in DataFrame path; suggested stronger UUID to pandas test | Applied both suggestions; replied on the review threads | early PR iterations (pre-C++ rewrite) |
| 2026-07-16 | (nudge) ready for another look | @mentioned @rok asking for re-review | - |
| 2026-07-21 | @rok: prefer C++ UUID conversion over Python compat layer ([rok/arrow#55](https://github.com/rok/arrow/pull/55)); asked if I could continue that way | Agreed (2026-07-23); rewrote approach around the proposal | leading into [`f71b3a2`](https://github.com/apache/arrow/pull/50325/commits/f71b3a2) |
| 2026-07-27 | (status) integrated proposal; round-trip passing; finishing last pass | Posted update thanking @rok for the proposal | [`f71b3a2`](https://github.com/apache/arrow/pull/50325/commits/f71b3a2) |
| 2026-07-29 | @pitrou: factor UUID support into existing `helpers.cc` | Moved construction into helpers | [`352cafd`](https://github.com/apache/arrow/pull/50325/commits/352cafd) |
| 2026-08-05 | @rok: "potential performance improvement... otherwise looks pretty good" - reuse empty kwargs instead of per-value dict in `helpers.cc` | Implemented reuse + shared empty tuple constant | [`ff16379`](https://github.com/apache/arrow/pull/50325/commits/ff163797f2b635d43dd4b5584698cee07c0d5bf9), [`2f35759`](https://github.com/apache/arrow/pull/50325/commits/2f35759533ba83986d9d3938b45eb96186929574) |
| 2026-08-10 | @rok: almost there; `to_numpy(zero_copy_only=False)[0]` must be `bytes` (storage type). Bot: `awaiting changes` | Fixed lint + NumPy storage behavior; asked @rok to re-run flaky macOS 15-intel MinIO job | [`834be3a`](https://github.com/apache/arrow/pull/50325/commits/834be3a), [`a4f309b`](https://github.com/apache/arrow/pull/50325/commits/a4f309b) |
| 2026-08-11 | @rok **APPROVED** ("LGTM", waiting on @AlenkaF / @pitrou). Bot: `awaiting merge`. @AlenkaF **APPROVED** (happy to see C++ solution merge) | Thanked @AlenkaF and @rok; credited [rok/arrow#55](https://github.com/rok/arrow/pull/55) | - |

---

## Learnings & Reflections

### Technical Skills Gained

- How PyArrow bridges Arrow extension types to pandas/NumPy: storage type vs logical type, and why falling back to `fixed_size_binary(16)` silently returns `bytes`.
- Building and iterating on Arrow's Python/C++ boundary (`arrow_to_pandas.cc`, `helpers.cc`) instead of staying only in Cython/Python.
- Reading maintainer proposals as first-class design input: integrating [rok/arrow#55](https://github.com/rok/arrow/pull/55) was faster and cleaner than defending the first Python-layer approach.
- Micro-efficiency in CPython API usage (reuse empty kwargs / borrowed empty tuple) when converting large arrays element-by-element.

### Challenges Overcome

- Initial mental model ("Python 3.14 regression") was wrong; reproduction on 3.10-3.14 showed a missing feature path exposed by newer downstream tests.
- First PR approach was acceptable as a spike, but maintainers correctly pushed for C++ conversion; rewriting mid-review without losing test coverage took discipline.
- Review loops spanned pandas reshape quirks, helper factoring, allocation reuse, and NumPy semantics - each needed a concrete follow-up commit, not just agreement in chat.

### What I'd Do Differently Next Time

- After root-causing a bridge bug, ask earlier whether the project prefers fixing it in C++ vs Python, instead of investing hard in the first matching in-repo Python pattern.
- Keep the PR description synchronized with approach changes (the body still described `to_pandas_dtype()` long after the C++ rewrite; updated in Phase IV cleanup).
- Treat "extension type to NumPy should follow storage" as a checklist item up front whenever adding logical-type conversions.

### Teachable insight for future cohorts

If your first patch matches a nearby Python helper but a maintainer sketches a C++ path, switch early. In mature projects like Arrow, the Python layer often exists for convenience; the durable fix usually lives next to the conversion engine. Document that approach change in your contribution log so Phase II and Phase IV stay consistent for graders and for your future self.

---

## Resources Used

- [apache/arrow#50312 - FIXED_LEN_BYTE_ARRAY fails to cast to UUID on Python 3.14](https://github.com/apache/arrow/issues/50312)
- [apache/arrow#50325 - PR: Fix UUID extension type round-trip to pandas](https://github.com/apache/arrow/pull/50325)
- [rok/arrow#55 - C++ UUID conversion proposal](https://github.com/rok/arrow/pull/55)
- [pandas-dev/pandas#65647 - upstream UUID Parquet tests](https://github.com/pandas-dev/pandas/pull/65647)
- [My fork: parker-cassar/arrow](https://github.com/parker-cassar/arrow)
- [Arrow Python development docs](https://arrow.apache.org/docs/developers/python/index.html)
- [Arrow contributing guide](https://github.com/apache/arrow/blob/main/CONTRIBUTING.md)
- [Arrow PR template](https://github.com/apache/arrow/blob/main/.github/pull_request_template.md)
