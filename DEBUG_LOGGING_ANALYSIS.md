# Debug Logging Analysis & Recommendations

## Problem Summary

After implementing comprehensive debug diagnostics for Unity 6 format investigation, we discovered:

- **Debug-only logging produces EMPTY log** (no entries)
- **Warning + Debug produces 34.5 MB log** with 110,064+ entries
- Log is too large to be practical for analysis

## Root Cause Analysis

### Log Entry Distribution (34.5 MB log)

```
42,372 Debug entries   (38.5%) - hex dumps, byte diagnostics
32,765 Warning entries (29.8%) - error messages
29,862 Info entries    (27.1%) - loading progress
 5,065 Error entries   (4.6%)  - critical failures
```

### Why Debug-Only Produces Empty Log

The Debug logging code has two main patterns:

**Pattern 1: Unity 6 Conditional**

```csharp
if (version[0] >= 6000 && Logger.Flags.HasFlag(LoggerEvent.Debug))
{
    Logger.Debug($"Unity 6 diagnostic...");
}
```

- Only logs for Unity 6000+ files
- Marvel Snap IS Unity 6000.0.58f2, so this should work

**Pattern 2: Error-Triggered Diagnostics**

```csharp
catch (Exception ex)
{
    Logger.Warning($"Failed to parse..."); // Requires Warning flag
    if (Logger.Flags.HasFlag(LoggerEvent.Debug))
    {
        Logger.Debug($"Diagnostic...");   // Requires Debug flag
    }
}
```

- Debug dumps only trigger when errors occur
- Errors log Warning messages first
- **Without Warning flag enabled, error handlers don't run**
- **Therefore Debug diagnostics inside error handlers never execute**

**Pattern 3: Unconditional Debug Calls**

```csharp
Logger.Debug($"SerializedPass: numIndices = {numIndices}...");
```

- These check `Flags.HasFlag(LoggerEvent.Debug)` internally
- Should work with Debug-only flag
- But these are INSIDE try blocks that throw exceptions
- When exception thrown, we jump to catch block (which needs Warning)

### The Real Issue

**Most Debug diagnostics are in catch blocks that only execute when errors occur.**

When errors occur:

1. Exception thrown
2. Catch block executes
3. `Logger.Warning()` is called → **requires Warning flag**
4. `Logger.Debug()` is called → requires Debug flag

**If Warning flag is NOT set:**

- Warning log call returns early (no output)
- Debug log call still executes
- BUT: The entire catch block only runs if code threw exception
- Many exceptions may be silently caught at higher levels

**Why 34.5 MB with Warning + Debug:**

- Every shader that fails parsing triggers:
  - 1 Warning message (~100 bytes)
  - 1-3 Debug dumps (128-256 bytes each = 512+ bytes)
- With 2,387 shader-related Debug entries × ~1KB avg = **~2.4 MB just for shader dumps**
- Plus TypeTree string failures, MonoBehaviour parsing, etc.
- **Total: 42,372 Debug entries × ~600 bytes avg = ~25 MB of Debug output**

## Solution Options

### Option 1: Reduce Debug Output Volume (RECOMMENDED)

**Changes:**

1. Reduce `DumpBytes()` default from 128/256 bytes to **32-64 bytes**
2. Only log **first 3 and last 3** items in loops (not all items)
3. Only log **suspicious values** (numIndices > 100 or < 0)
4. Keep failure diagnostics verbose (still helpful for format investigation)

**Expected Result:** 34.5 MB → ~8-10 MB

### Option 2: Make Debug Independent of Warning

**Changes:**

1. Add standalone Debug logging outside try-catch blocks
2. Log successful parsing with Debug flag (not just failures)
3. Add "Unity 6 shader parsing started" Debug entries

**Expected Result:** Debug-only will produce output, but still large volume

### Option 3: Add Sampling Controls

**Changes:**

1. Add `Logger.DebugSampleRate` setting (default: 10 = log 1 in 10)
2. Only dump diagnostics for sampled objects
3. Keep all Warning/Error messages

**Expected Result:** Configurable volume reduction

### Option 4: Add Debug Sub-Flags

**Changes:**

```csharp
public enum LoggerEvent
{
    Debug = 2,
    DebugShaders = 32,      // Shader-specific debug
    DebugTypeTree = 64,     // TypeTree-specific debug
    DebugBytes = 128,       // Hex dump diagnostics
}
```

**Expected Result:** Fine-grained control, but complex implementation

## Recommended Approach

**Implement Option 1 (Volume Reduction) immediately:**

1. **Reduce dump sizes:**

   - DumpBytes(128) → DumpBytes(64)
   - DumpBytes(256) → DumpBytes(128)
   - Only for failure diagnostics

2. **Add smart filtering:**

   ```csharp
   // Only log suspicious or boundary values
   if (numIndices > 100 || numIndices < 0)
   {
       Logger.Debug($"SUSPICIOUS: numIndices = {numIndices}");
   }

   // Only log first/last items in loops
   if (i < 3 || i >= numIndices - 3)
   {
       Logger.Debug($"NameIndex[{i}]: '{name}' = {index}");
   }
   ```

3. **Keep verbose failure dumps:**

   - Error diagnostics are most valuable
   - Worth the extra bytes when something fails

4. **Document Warning + Debug requirement:**
   - Update DEBUG_LOGGING.md
   - Explain that Debug diagnostics trigger on errors
   - Errors require Warning level to be logged

## Implementation Priority

1. ✅ **HIGH**: Reduce DumpBytes() sizes (64 bytes for success, 128 for failures)
2. ✅ **HIGH**: Filter loop logging (first 3 + last 3 only)
3. ✅ **HIGH**: Only log suspicious values
4. ⏳ **MEDIUM**: Update DEBUG_LOGGING.md with findings
5. ⏳ **LOW**: Consider sampling for future if still too large

## Expected Results After Changes

**Current:** 34.5 MB, 110,064 entries
**After reduction:** ~8-10 MB, ~50,000 entries
**Breakdown:**

- Debug: 42,372 → ~15,000 entries (filtering loops, reducing dump sizes)
- Warning: 32,765 (unchanged)
- Info: 29,862 (unchanged)
- Error: 5,065 (unchanged)

**Debug reduction calculation:**

- Shader loops: Log 1,000 shaders × 6 items avg → Log 6 items (first/last 3) = 83% reduction
- Dump sizes: 128/256 bytes → 64/128 bytes = 50% reduction
- Total Debug size: ~25 MB → ~6-8 MB

## Why Debug-Only Still Won't Work

Even after volume reduction, **Debug-only logging will remain limited** because:

1. Most interesting Debug output happens in error scenarios
2. Error scenarios require Warning-level logging to trigger
3. Successful parsing has minimal Debug output (by design)

**This is actually correct behavior:**

- Debug diagnostics are for **investigating format problems**
- Format problems manifest as **parsing errors**
- Parsing errors are logged as **Warnings**
- Therefore: Debug requires Warning to be useful

**User should use:** `Logger.Flags = LoggerEvent.Warning | LoggerEvent.Debug`

This is the intended usage pattern for format investigation.
