# github-contribution-log

# Contribution [1]: PyArrow fails to cast UUID FIXED_LEN_BYTE_ARRAY back to `uuid.UUID` on Python 3.14

**Contribution Number:** 1
**Student:** Parker Cassar
**Issue:** https://github.com/apache/arrow/issues/50312
**Fork:** https://github.com/parker-cassar/arrow
**Status:** Phase II — Reproduce and Plan

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

**Branch:** [`gh-50312-uuid-pandas-roundtrip`](https://github.com/parker-cassar/arrow/tree/gh-50312-uuid-pandas-roundtrip) in my fork (named after issue #50312).

**Setup approach:** Followed upstream docs rather than a dev container. Read [CONTRIBUTING.md](https://github.com/apache/arrow/blob/main/CONTRIBUTING.md) for the issue/PR workflow, then the [Python development guide](https://arrow.apache.org/docs/developers/python/index.html) for the CMake + editable-install build. Inspected [`.github/workflows/python.yml`](https://github.com/apache/arrow/blob/main/.github/workflows/python.yml) to see how CI runs Python tests (`pytest python/pyarrow/tests/`) across matrix Python versions — I'll mirror that locally before opening a PR.

**Steps taken:**

1. Forked `apache/arrow` → [parker-cassar/arrow](https://github.com/parker-cassar/arrow)
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
| Issue says 3.13 returns `uuid.UUID` but my 3.13 venv also returned `bytes` with PyArrow 24.0.0 | Ran intermediate-type checks (`to_pylist()` vs `to_pandas()`) to isolate the failure to the pandas bridge, not Parquet I/O — see findings below |
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
4. **Expected:** `<class 'uuid.UUID'>` — the value written should roundtrip as a Python UUID object.
5. **Actual (Python 3.14):** `<class 'bytes'>` — raw 16-byte blob instead of a UUID.
6. To confirm the failure is in the pandas bridge (not Parquet I/O), add these checks:
   ```python
   print(read_table.column("id").type)                    # extension<arrow.uuid>
   print(type(read_table.column("id").to_pylist()[0]))    # uuid.UUID (works)
   print(type(read_table.to_pandas().loc[0, "id"]))        # bytes (broken)
   ```
7. Repeat steps 1–6 on Python 3.13 as a control. Parquet schema and `to_pylist()` behave identically; `to_pandas()` returns `bytes` on both with PyArrow 24.0.0.

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
- `2328b6e` (2024-08-26) — [GH-15058](https://github.com/apache/arrow/pull/37298): native UUID extension type added (`UuidType`, `UuidScalar.as_py()`)
- `75acf37` (2025-04-21) — [GH-43807](https://github.com/apache/arrow/pull/45866): Parquet read/write support for UUID added
- Neither commit added `to_pandas_dtype()` on `UuidType`. The gap has existed since UUID support landed; Python 3.14 / pandas 3.0 likely exposed it because downstream test suites (e.g. `pandas-dev/pandas#65647`) now assert `to_pandas()` roundtrips.

**Relevant code paths identified:**
- `python/pyarrow/types.pxi` — `UuidType`, `UuidScalar`, `UuidArray` definitions
- `python/pyarrow/scalar.pxi` — `UuidScalar.as_py()` correctly returns `UUID(bytes=...)`
- `python/pyarrow/array.pxi` — `_array_like_to_pandas()` extension type branch
- `python/pyarrow/pandas_compat.py` — `_get_extension_dtypes()` for table-level conversion
- `python/pyarrow/public-api.pxi` — registers `arrow.uuid` extension on Parquet read
- `python/pyarrow/tests/parquet/test_data_types.py` — existing `test_uuid_extension_type()` (roundtrip at Arrow level, no `to_pandas` assertion)

---

## Solution Approach

### Analysis

The bug is not in Parquet serialization — UUID data is stored and read back as `extension<arrow.uuid>` with `fixed_size_binary(16)` storage in both Python versions. The regression is in how PyArrow converts that extension array to a pandas Series via `to_pandas()`.

The conversion logic in `_array_like_to_pandas` checks whether an extension type provides a pandas dtype via `to_pandas_dtype()`. If the returned dtype has `__from_arrow__`, it uses that for a typed conversion. `UuidType` has no such mapping, so PyArrow delegates to the generic C++ converter, which materializes the storage bytes rather than calling `UuidScalar.as_py()`.

The fix should ensure that when `to_pandas()` encounters an `extension<arrow.uuid>` column, the result contains `uuid.UUID` objects — matching what `to_pylist()` already produces.

### Proposed Solution

Teach the `to_pandas()` path to handle `UuidType` the same way the native Arrow path already does. Two viable approaches:

1. **Implement `to_pandas_dtype()` on `UuidType`** that returns a pandas-compatible dtype with `__from_arrow__`, converting via `UuidScalar.as_py()`. This follows the pattern used by other extension types (e.g. `ExampleUuidType` in the test suite).
2. **Add a special case in `_array_like_to_pandas`** for `UuidType` that converts through `to_pylist()` or iterates `UuidScalar.as_py()` before building the Series. Simpler but less extensible.

Option 1 is preferred — it matches existing PyArrow conventions and will also benefit `types_mapper`-based table conversions in `pandas_compat.py`.

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** When reading a Parquet file containing UUIDs, `table.to_pandas()` must return `uuid.UUID` objects, not raw `bytes`. The Arrow layer already handles this correctly; the pandas bridge does not.

**Match:** The closest in-repo analogue is `MyCustomIntegerType` in `python/pyarrow/tests/test_pandas.py`, which implements `to_pandas_dtype()` returning `pd.Int64Dtype()` — a pandas extension dtype with `__from_arrow__`. That test (`test_conversion_extensiontype_to_extensionarray`) proves the exact pattern I need: when `to_pandas_dtype()` returns a dtype with `__from_arrow__`, `_array_like_to_pandas` takes the typed path instead of falling through to C++. A secondary analogue is `PeriodTypeWithToPandasDtype` in `test_extension_type.py`, which adds `to_pandas_dtype()` to an extension type returning `pd.PeriodDtype(freq=...)`. For UUID, I'll follow the same pattern — either wire up a pandas UUID extension dtype (if available in pandas 3.0) or provide a small helper dtype whose `__from_arrow__` delegates to `UuidScalar.as_py()`.

**Edge cases to handle in the fix:**
- **Null UUIDs** — a nullable UUID column should roundtrip as `None`, not empty bytes
- **Empty tables** — `to_pandas()` on a zero-row UUID column should not crash
- **`types_mapper` override** — if a caller passes a custom mapper, it should take precedence over the default UUID mapping
- **Parquet with `arrow_extensions_enabled=False`** — reading as plain `fixed_size_binary(16)` is expected to stay as bytes; only the extension-typed path needs UUID conversion

**Plan:**
1. Add a failing regression test in `python/pyarrow/tests/parquet/test_data_types.py` (or `test_extension_type.py`) that writes a UUID column to Parquet, reads it back, calls `to_pandas()`, and asserts `isinstance(result, uuid.UUID)`.
2. Implement `to_pandas_dtype()` on `UuidType` (or a dedicated pandas extension dtype helper) that provides `__from_arrow__` returning `uuid.UUID` objects.
3. Verify the fix on Python 3.14 and confirm no regression on Python 3.13.
4. Run existing UUID and Parquet test suites: `pytest python/pyarrow/tests/parquet/test_data_types.py python/pyarrow/tests/test_extension_type.py`.

**Implement:** [Link to branch/commits as work begins in Phase III]

**Review:** Confirm the change follows Arrow's contribution guidelines — prefix PR title with `GH-50312: [Python][Parquet]`, include a test, no unrelated changes.

**Evaluate:** The regression test passes on Python 3.14. Existing `test_uuid_extension_type` and related Parquet tests still pass. Manual repro script from the issue returns `<class 'uuid.UUID'>`.

---

## Testing Strategy

### Unit Tests

- [ ] Parquet UUID roundtrip via `to_pandas()` returns `uuid.UUID` on Python 3.14
- [ ] `UuidType.to_pandas_dtype()` (or equivalent) produces a dtype with working `__from_arrow__`
- [ ] Null UUID values roundtrip as `None`, not as empty bytes

### Integration Tests

- [ ] Full issue repro script passes end-to-end on Python 3.14
- [ ] Existing `test_uuid_extension_type` in `parquet/test_data_types.py` still passes (no regression)

### Manual Testing

Run the issue's repro script on Python 3.13 and 3.14 before and after the fix. Confirm `type(result_df.loc[0, "id"])` is `uuid.UUID` on both. Also spot-check `to_pylist()` still returns `uuid.UUID` (should be unchanged).

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
- Confirmed Parquet roundtrip preserves `extension<arrow.uuid>` — the bug is in `to_pandas()`, not Parquet I/O
- Traced the failure to `_array_like_to_pandas()` in `array.pxi` where `UuidType` lacks `to_pandas_dtype()`
- Used `git log` to date the gap: UUID + Parquet support landed in 2024–2025 without a pandas conversion path
- Identified `MyCustomIntegerType.to_pandas_dtype()` in `test_pandas.py` as the pattern to follow
- Drafted fix plan with edge cases (nulls, empty tables, `types_mapper`, `arrow_extensions_enabled=False`)

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

- [apache/arrow#50312 — FIXED_LEN_BYTE_ARRAY fails to cast to UUID on Python 3.14](https://github.com/apache/arrow/issues/50312)
- [pandas-dev/pandas#65647 — upstream UUID Parquet tests](https://github.com/pandas-dev/pandas/pull/65647)
- [My fork: parker-cassar/arrow](https://github.com/parker-cassar/arrow)
- [Arrow Python development docs](https://arrow.apache.org/docs/developers/python/index.html)
- [Arrow contributing guide](https://github.com/apache/arrow/blob/main/CONTRIBUTING.md)
