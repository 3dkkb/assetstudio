# Debug Logging Guide

**Purpose**: Enable detailed diagnostic logging to investigate Unity format issues

---

## ⚠️ Important: Debug Requires Warning Level

**Debug logging is designed to provide diagnostics for error conditions.**

Most Debug output appears in error handlers that only execute when parsing fails:

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

**To see Debug diagnostics, you MUST enable Warning level:**

```csharp
// ✅ CORRECT - Debug diagnostics will appear
Logger.Flags = LoggerEvent.Warning | LoggerEvent.Debug;

// ❌ WRONG - Debug-only produces minimal/empty log
Logger.Flags = LoggerEvent.Debug;
```

**This is intentional:** Debug logging investigates format problems, which manifest as parsing errors (Warnings).

---

## Quick Start

### Enable Debug Logging in GUI

Edit `AssetStudio.GUI/GUILogger.cs` or set in code:

```csharp
// For format investigation (Unity 6 issues)
Logger.Flags = LoggerEvent.Warning | LoggerEvent.Debug;

// Or enable everything
Logger.Flags = LoggerEvent.All;
```

### Enable Debug Logging in CLI

Command line argument (if implemented):

```bash
AssetStudio.CLI.exe input output --debug
```

Or set in `Program.cs`:

```csharp
Logger.Flags = LoggerEvent.Warning | LoggerEvent.Debug;
```

---

## ⚡ Log Volume Management

**Debug logging is VERY verbose.** Expect large log files:

- **Without volume controls**: 30-40 MB for typical game
- **With volume controls (v2.3.2+)**: 8-12 MB for typical game

### Volume Control Features (v2.3.2+)

1. **Reduced dump sizes**: 64-128 bytes instead of 128-256 bytes
2. **Loop sampling**: Only first 3 and last 3 items logged
3. **Smart filtering**: Only logs suspicious values (e.g., numIndices > 100)
4. **TypeTree sampling**: 1 in 50 string reads logged (root level always logged)

### Tips for Managing Large Logs

**Filter with PowerShell:**

```powershell
# Show only errors
Get-Content log.txt | Select-String "\[Error\]"

# Show shader-related debug entries
Get-Content log.txt | Select-String "\[Debug\].*Shader"

# Count log level distribution
Get-Content log.txt | Select-String "\[(Debug|Warning|Error|Info)\]" |
    ForEach-Object { if ($_ -match '\]\[(.*?)\]') { $matches[1] } } |
    Group-Object | Sort-Object Count -Descending

# Show first 100 debug entries
Get-Content log.txt | Select-String "\[Debug\]" | Select-Object -First 100
```

**Use external tools:**

- **LogExpert** (Windows) - Fast log viewer with filtering
- **Notepad++** with large file support
- **VS Code** with line limit warnings disabled

---

## What Debug Logging Shows

### 1. Shader Format Diagnostics

**SerializedPass parsing** (Unity 6 issues):

```
[Debug] SerializedPass SUSPICIOUS: numIndices = 1953066613 at position 0x00012A40
[Pre-NameIndices]
Position: 0x00012A40 (76352), Remaining: 296520 bytes
Hex: 03 00 00 00 5F 53 72 63 42 6C 65 6E 64 00 00 00
     00 00 00 00 5F 44 73 74 42 6C 65 6E 64 00 00 00
     01 00 00 00 5F 5A 57 72 69 74 65 00 00 00 00 00
As Int32s: 3, 1668445023, 1818386285, 0, 1, 1953067103, 7561330, 0,

[Debug] SerializedPass: numIndices = 3, position = 0x00012A44
[Debug]   NameIndex[0]: '_SrcBlend' = 0
[Debug]   NameIndex[1]: '_DstBlend' = 0
[Debug]   NameIndex[2]: '_ZWrite' = 0
```

**On failure**:

```
[Warning] Failed to parse SerializedPass (Unity 6000.0.58): String length 1818168615 exceeds...
[Debug] Unity 6 SerializedPass parse failure diagnostic:
[SerializedPass failure context]
Position: 0x00012A68, Remaining: 296480 bytes
Hex: 6C 6C 20 6C 69 67 68 74 73 00 00 00 03 00 00 00
     5F 53 72 63 42 6C 65 6E 64 00 00 00 00 00 00 00
     ...
As Int32s: 1819243372, 7561330, 3, 1668445023, ...
```

### 2. TypeTree String Read Diagnostics

**MonoBehaviour/UI_en issues**:

```
[Debug] TypeTree reading string field 'm_Name' at position 0x00004520
[Debug] [Pre-read: m_Name]
Position: 0x00004520 (17696), Remaining: 8432 bytes
Hex: 07 00 00 00 55 49 5F 65 6E 00 00 00
     (length: 7, string: "UI_en")

[Debug] TypeTree string 'm_Name' = 'UI_en'
```

**On failure**:

```
[Warning] Error reading TypeTree field 'm_Description' (type: string): String length 1953066613 exceeds...
[Debug] TypeTree field read failure diagnostic:
[Field 'm_Description' failure]
Position: 0x00004650, Remaining: 8102 bytes
Hex: 75 73 65 72 64 61 74 61 00 00 00 00 ...
As Int32s: 1970434421, 1684108385, 0, ...
```

### 3. EndianBinaryReader Byte Dumps

The `DumpBytes()` method shows:

- **Position**: Current stream position (hex and decimal)
- **Remaining**: Bytes left to read
- **Hex dump**: Up to 64 bytes in hex (16 bytes per line)
- **Int32 interpretation**: First 32 bytes interpreted as Int32 values

**Example**:

```
Position: 0x00012A40 (76352), Remaining: 296520 bytes
Hex: 03 00 00 00 5F 53 72 63 42 6C 65 6E 64 00 00 00
     00 00 00 00 5F 44 73 74 42 6C 65 6E 64 00 00 00
     01 00 00 00 5F 5A 57 72 69 74 65 00 00 00 00 00
     00 00 00 00 02 00 00 00 43 75 6C 6C 00 00 00 00
As Int32s: 3, 1668445023, 1818386285, 0, 1, 1953067103, 7561330, 0,
```

---

## Use Cases

### 1. Finding Unity 6 Format Changes

**Problem**: Unity 6 shaders fail at `numIndices` read

**Steps**:

1. Enable Debug logging
2. Load Unity 6 game files
3. Look for `[Debug] SerializedPass (Unity 6000.0.58) before numIndices read`
4. Compare hex dump with Unity 2022 (working version)
5. Identify new fields or changed field order

**What to look for**:

- Extra Int32 values before expected field
- Different array lengths
- Changed alignment patterns

### 2. Comparing Unity Versions

**Load same asset from two Unity versions**:

```
Unity 2022.3:
Position: 0x1000, Remaining: 5000
Hex: 02 00 00 00 5F 43 6F 6C 6F 72 ...
As Int32s: 2, 1869504351, ...

Unity 6000.0:
Position: 0x1000, Remaining: 5120  ← More data!
Hex: 01 00 00 00 02 00 00 00 5F 43 6F 6C 6F 72 ...
         ↑ NEW FIELD?
As Int32s: 1, 2, 1869504351, ...
```

Conclusion: Unity 6 added a 4-byte field before the string count!

### 3. Investigating String Read Failures

**Problem**: "String length 1818168615 exceeds remaining bytes"

**Hex interpretation**:

```
Hex: 6C 6C 20 6C  ← This is being read as length
      l  l     l
```

**Realization**: We're reading ASCII text as an Int32!

- `6C 6C 20 6C` in ASCII = "ll l"
- As Int32 (little-endian) = 0x6C206C6C = 1818168615

**Conclusion**: We missed a field, so stream position is wrong.

### 4. Finding Missing Fields

**Pattern in Debug log**:

```
[Debug] Reading field A: OK at 0x1000
[Debug] Reading field B: OK at 0x1020
[Debug] Reading field C: FAIL at 0x1040
  Hex at failure: 01 00 00 00 05 00 00 00 ...
  Expected string, got: Int32(1), Int32(5), ...
```

**Solution**: Unity added 1-2 Int32 fields between B and C in this version.

---

## Debug Output File

Debug logs go to `log.txt` in the application directory.

**File location**:

- GUI: `AssetStudio.GUI\bin\Release\net10.0-windows\log.txt`
- CLI: `AssetStudio.CLI\bin\Release\net10.0-windows\log.txt`

**Log format**:

```
[11/19/2025 8:45:32 PM][Debug] message
[11/19/2025 8:45:32 PM][Warning] message
```

**Tip**: Use PowerShell to filter:

```powershell
# Show only Debug logs
Select-String -Path log.txt -Pattern "\[Debug\]"

# Show Unity 6 shader diagnostics
Select-String -Path log.txt -Pattern "Unity 6.*SerializedPass" -Context 5,10

# Extract hex dumps
Select-String -Path log.txt -Pattern "Hex:" -Context 0,4
```

---

## Performance Impact

**Debug logging is VERBOSE** - it will:

- ✅ Slow down loading significantly (2-10x slower)
- ✅ Generate HUGE log files (hundreds of MB)
- ✅ Show diagnostic info for EVERY shader/object

**Recommendations**:

- ❌ Don't enable Debug for production use
- ✅ Enable only when investigating specific issues
- ✅ Load small test files (1-100 assets)
- ✅ Disable after collecting needed diagnostics

**Selective debugging**:

```csharp
// Only log for Unity 6
if (version[0] >= 6000 && Logger.Flags.HasFlag(LoggerEvent.Debug))
{
    Logger.Debug(...);
}
```

---

## Reading Hex Dumps

### Common Patterns

**Int32 (little-endian)**:

```
03 00 00 00  →  3
FF 00 00 00  →  255
00 01 00 00  →  256
```

**String (length + UTF-8 + alignment)**:

```
07 00 00 00 55 49 5F 65 6E 00 00 00
↑ length=7  U  I  _  e  n  ↑ padding to 4-byte align
```

**Array (count + elements)**:

```
02 00 00 00  01 00 00 00  02 00 00 00
↑ count=2    ↑ element[0]=1  ↑ element[1]=2
```

**Boolean + padding**:

```
01 00 00 00  →  true (with 3 bytes padding)
```

### Spotting Format Issues

**❌ Bad string read** (reading data as length):

```
Position: 0x1000
Hex: 6C 6C 20 6C ...
As Int32s: 1818168615, ...  ← Huge number!
            ↑ This is ASCII text, not a length!
```

**✅ Good string read**:

```
Position: 0x1000
Hex: 05 00 00 00 48 65 6C 6C 6F 00 00 00
      ↑ 5         H  e  l  l  o  ↑ padding
```

**Missing alignment**:

```
Before align: Position: 0x1005 (odd)
After align:  Position: 0x1008 (next 4-byte boundary)
```

---

## Troubleshooting

### Debug logs not appearing

**Check**:

1. `Logger.Flags` includes `LoggerEvent.Debug`
2. File logging is enabled: `Logger.FileLogging = true`
3. Check `log.txt` in application directory

### Log file too large

**Solutions**:

- Load fewer files
- Filter specific assets
- Use Verbose instead of Debug for less data
- Clear `log.txt` between runs

### Can't find specific diagnostic

**Search patterns**:

```powershell
# Shader diagnostics
Select-String -Path log.txt -Pattern "SerializedPass.*before numIndices"

# String failures
Select-String -Path log.txt -Pattern "String length.*exceeds"

# Unity 6 issues
Select-String -Path log.txt -Pattern "Unity 6" | Select-Object -First 50
```

---

## API Reference

### EndianBinaryReader.DumpBytes()

```csharp
/// <summary>
/// Dumps bytes at current position for diagnostic purposes (DEBUG level)
/// </summary>
/// <param name="count">Number of bytes to dump (default: 128)</param>
/// <param name="context">Context string for the dump (e.g., "Pre-NameIndices")</param>
/// <returns>Formatted string with hex dump and int interpretations</returns>
public string DumpBytes(int count = 128, string context = "")
```

**Usage**:

```csharp
if (Logger.Flags.HasFlag(LoggerEvent.Debug))
{
    Logger.Debug(reader.DumpBytes(256, "Before reading problematic field"));
}
```

### Logger.Debug()

```csharp
public static void Debug(string message)
```

**Controlled by**:

```csharp
Logger.Flags = LoggerEvent.Debug;  // Enable
Logger.Flags &= ~LoggerEvent.Debug; // Disable
```

---

## Next Steps

1. **Enable Debug logging**
2. **Load Unity 6 game file** (Marvel Snap recommended)
3. **Review log.txt** for diagnostic output
4. **Compare hex dumps** between successful and failed parses
5. **Identify missing fields** or format changes
6. **Update code** with version-specific logic
7. **Test and validate** with multiple Unity 6 games

**Good luck investigating! 🔍**
