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
This issue acts as a good-first enhancement issue. It proposed using "disk size" units to define the chunk, instead of dimensions.

### Expected Behavior
`zarr.zeros(..., chunks="100MB", ...)` will chunk in a way that each chunk is approximately 100 MB, with zarr automatically computing a sensible chunk shape from the array's shape and dtype.

### Current Behavior
Only a dimension shape is accepted — e.g. `zarr.zeros(..., chunks=(100, 100), ...)`. Passing a byte-size string like `"100MB"` causes a confusing, misleading error:

```
ValueError: chunks has 5 dimensions but shape has 2 dimensions
```

The string `"100MB"` is silently iterated as 5 characters (`'1','0','0','M','B'`), which zarr interprets as a 5-dimensional chunk spec, not a byte size.

### Affected Components

- `src/zarr/core/array.py` — `init_array()`, the internal function where `chunks` is dispatched
- `src/zarr/core/chunk_grids.py` — `normalize_chunks_nd()`, where the string falls through to and breaks; and `guess_chunks()`, the heuristic that already accepts `max_bytes`
- `src/zarr/core/common.py` — `ChunksLike` type alias; home for the new `parse_byte_size` helper
- All public API entry points: `src/zarr/api/synchronous.py`, `src/zarr/api/asynchronous.py`, `src/zarr/core/group.py`

---

## Reproduction Process

### Environment Setup

```bash
# Clone your fork and add the upstream remote
git clone git@github.com:<your-username>/zarr-python.git
cd zarr-python
git remote add upstream git@github.com:zarr-developers/zarr-python.git

# Install hatch (the project's environment manager)
pip install hatch

# Verify the dev environment works
hatch -e test.py3.12-optional run python -c "import zarr; print(zarr.__version__)"
```

### Steps to Reproduce

**Step 1:** Open a Python shell inside the hatch dev environment:
```bash
hatch -e test.py3.12-optional run python
```

**Step 2:** Run the following:
```python
import zarr

# Attempt byte-size string — this is the feature gap
try:
    z = zarr.zeros((1000, 1000), chunks="100MB", dtype="float64")
    print("SUCCESS:", z.chunks)
except Exception as e:
    print("FAILS:", repr(e))

# "auto" works fine today
z = zarr.zeros((1000, 1000), chunks="auto", dtype="float64")
print("auto chunks:", z.chunks)

# The low-level primitive already handles max_bytes — the plumbing is there
from zarr.core.chunk_grids import guess_chunks
result = guess_chunks((1000, 1000), typesize=8, max_bytes=100 * 1024 * 1024)
print("guess_chunks(max_bytes=100MB):", result)
```

**Observed result:**
```
FAILS: ValueError('chunks has 5 dimensions but shape has 2 dimensions')
auto chunks: (256, 256)
guess_chunks(max_bytes=100MB): (ChunksTuple with shape (1000, 1000))
```

The key observation: `guess_chunks` with `max_bytes` already works perfectly — the heuristic layer is complete. Only the string-parsing and routing layer is missing.

### My Findings

- The string `"100MB"` bypasses the `"auto"` check at `array.py:4422` and is passed directly to `normalize_chunks_nd()` at `array.py:4426`, which treats it as an iterable of characters.
- `guess_chunks()` in `chunk_grids.py:804` already accepts a `max_bytes: int | None` parameter — the hard work is done.
- No byte-size string parser exists anywhere in the codebase yet.
- The existing `parse_*` helper functions in `src/zarr/core/common.py` (e.g., `parse_int`, `parse_bool`, `parse_shapelike`) define the exact pattern a new `parse_byte_size` function should follow.

---

## Solution Approach

### Analysis

**Root cause — `init_array()` in `src/zarr/core/array.py:4422-4426`:**

```python
if chunks == "auto":
    max_bytes = None if shards is None else SHARDED_INNER_CHUNK_MAX_BYTES
    chunks_normalized = guess_chunks(shape_parsed, item_size, max_bytes=max_bytes)
else:
    # ← any string other than "auto" lands here
    chunks_normalized = normalize_chunks_nd(chunks, shape_parsed)
```

`normalize_chunks_nd` has no string handling — it calls `len(chunks)` on the string (giving character count = fake dimension count), then raises a dimension mismatch error. The fix is a new `elif isinstance(chunks, str)` branch that parses the string and routes to `guess_chunks(..., max_bytes=N)`.

### Proposed Solution

Add a `parse_byte_size(data: str) -> int` helper to `src/zarr/core/common.py` that converts strings like `"100MB"` or `"1GiB"` into an integer number of bytes, using 1024-based arithmetic to match zarr's existing internal constants. Then intercept byte-size strings in `init_array()` before they reach `normalize_chunks_nd()`, and route them through `guess_chunks(..., max_bytes=parse_byte_size(chunks))`.

---

## Implementation Plan

### Understand

zarr-python accepts `chunks="auto"` to let the library guess a good chunk shape. This issue asks for `chunks="100MB"` to mean "guess a chunk shape, but constrain it to ~100 MB per chunk." The internal heuristic (`guess_chunks`) already supports a byte constraint — only the user-facing parsing layer is missing. The fix is small and localized to three places: a new parser function, a new branch in the dispatch logic, and type/docstring updates across the public API.

### Match

The existing pattern to follow is in `src/zarr/core/common.py` — all `parse_*` helpers (`parse_int` at line 220, `parse_bool` at line 214, `parse_shapelike` at line 178) take a broad input, validate it, raise `ValueError` with a clear message on failure, and return a plain Python primitive. The new `parse_byte_size` fits this pattern exactly.

The dispatch pattern to follow is the existing `chunks == "auto"` branch at `array.py:4422` — a new `elif isinstance(chunks, str)` branch sits immediately after it.

### Plan

1. **Add `parse_byte_size(data: str) -> int`** to `src/zarr/core/common.py` (after `parse_int` at line 220). Support `"B"`, `"KB"`, `"MB"`, `"GB"`, `"TB"` (and `"KiB"` / `"MiB"` / `"GiB"` aliases), case-insensitive, with optional space between number and unit. Use 1024-based math. Raise `ValueError` on unrecognised format.

2. **Extend the dispatch branch** in `init_array()` at `src/zarr/core/array.py:4422`:
   ```python
   if chunks == "auto":
       ...
   elif isinstance(chunks, str):          # NEW
       chunks_normalized = guess_chunks(
           shape_parsed, item_size, max_bytes=parse_byte_size(chunks)
       )
   else:
       chunks_normalized = normalize_chunks_nd(chunks, shape_parsed)
   ```

3. **Update type annotations** at every public entry point to widen `ChunksLike | Literal["auto"]` to `ChunksLike | Literal["auto"] | str`. Do **not** add `str` to the `ChunksLike` alias itself — keep the conversion at the boundary.

4. **Update docstrings** (numpydoc style) for `chunks` parameter at `init_array`, `zeros`, `ones`, `empty`, `full`, `create_array`, `Group.create_array`.

5. **Add changelog fragment:** `towncrier create` → feature, issue #270.

### Implement

_Branch: `fix-issue-270` — link to PR once opened_

### Review

Self-review checklist against `docs/contributing.md`:

- [ ] `hatch -e test.py3.12-optional run-coverage` passes at 100%
- [ ] `ruff check src tests` and `ruff format --check src tests` are clean
- [ ] `mypy src` passes (strict mode — verify the `str` annotation widening doesn't introduce errors)
- [ ] `prek run --all-files` passes (pre-commit hooks)
- [ ] All docstring examples are valid doctests
- [ ] PR description is written in my own words and explains the approach
- [ ] PR scope is limited to this feature only

### Evaluate

**Unit tests** for `parse_byte_size` in `tests/test_chunk_grids.py` (following the `Expect`/`ExpectFail` pattern in `tests/conftest.py`):

| Input | Expected |
|---|---|
| `"1B"` | `1` |
| `"1KB"` | `1024` |
| `"100MB"` | `104857600` |
| `"1GB"` | `1073741824` |
| `"1MiB"` | `1048576` |
| `"50 MB"` | `52428800` |
| `"1mb"` | `1048576` |
| `"100"` (no unit) | `ValueError` |
| `"MB"` (no number) | `ValueError` |
| `"1XB"` (unknown unit) | `ValueError` |

**Integration tests** in `tests/test_array.py`:

```python
@pytest.mark.parametrize("byte_str,dtype,shape", [
    ("100MB", "float64", (1000, 1000)),
    ("1MB",   "uint8",   (512, 512)),
    ("64MB",  "float32", (100, 100, 100)),
])
def test_chunks_byte_size_string(byte_str, dtype, shape):
    z = zarr.zeros(shape, chunks=byte_str, dtype=dtype)
    chunk_bytes = np.prod(z.chunks) * np.dtype(dtype).itemsize
    assert chunk_bytes <= parse_byte_size(byte_str)
    assert all(c > 0 for c in z.chunks)
    assert len(z.chunks) == len(shape)

def test_chunks_invalid_string_gives_clear_error():
    with pytest.raises(ValueError, match="not a valid byte size"):
        zarr.zeros((100, 100), chunks="100notaunit")
```

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
