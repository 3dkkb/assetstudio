# AssetStudio - AI Quick Reference Card

**Fast lookup for AI assistants working on AssetStudio**

---

## 🎯 Project Essentials

| Item                | Value                                        |
| ------------------- | -------------------------------------------- |
| **Language**        | C# (.NET 10)                                 |
| **Purpose**         | Unity asset extraction (reverse engineering) |
| **Unity Support**   | Unity 2.x → Unity 6 (6000.x)                 |
| **Current Version** | 2.3.1 (Nov 2025)                             |
| **Architecture**    | Multi-threaded, parallel processing          |
| **Main Branch**     | `main`                                       |

---

## 📁 Critical Files

```
AssetStudio/
├── AssetsManager.cs          # Main orchestrator (START HERE)
├── SerializedFile.cs         # Asset file parser
├── ObjectReader.cs           # Binary reader with version context
├── Classes/
│   ├── Shader.cs            # ⚠️ COMPLEX - Unity 6 issues
│   ├── Texture2D.cs         # Image assets
│   ├── Mesh.cs              # 3D models
│   └── Material.cs          # Material assets

AssetStudio.CLI/Program.cs    # CLI entry
AssetStudio.GUI/MainForm.cs   # GUI entry
AssetStudio.Utility/          # Export converters
```

---

## 🔧 Common Code Patterns

### Unity Version Checks

```csharp
// Major version (handles Unity 6 = 6000)
if (version[0] >= 6000)
{
    // Unity 6+ format
}

// Specific version
if (version[0] > 2020 || (version[0] == 2020 && version[1] >= 2))
{
    // 2020.2 and up
}

// Version array: [major, minor, patch, type]
// Unity 2021.3.10f1 → [2021, 3, 10, 1]
// Unity 6000.0.58f2 → [6000, 0, 58, 2]
```

### Reading Arrays

```csharp
int count = reader.ReadInt32();
var items = new List<MyType>(count);  // Preallocate
for (int i = 0; i < count; i++)
{
    items.Add(new MyType(reader));
}
```

### Alignment (CRITICAL)

```csharp
// After reading variable-length data (strings, arrays)
reader.AlignStream();  // Aligns to 4-byte boundary
```

### Error Handling (Graceful Degradation)

```csharp
public MyClass(ObjectReader reader) : base(reader)
{
    try
    {
        // Parse all fields
        m_Field1 = reader.ReadInt32();
        m_Field2 = reader.ReadAlignedString();
    }
    catch (Exception ex)
    {
        Logger.Warning($"Failed to parse {GetType().Name} (Unity {reader.version[0]}.{reader.version[1]}): {ex.Message}");
        // Object still accessible with partial data
    }
}
```

### PPtr (Unity Object References)

```csharp
// Reading reference
var shader = new PPtr<Shader>(reader);

// Resolving reference
var actualShader = shader.TryGet(assetsFile);
if (actualShader != null)
{
    // Use shader
}
```

---

## 🚨 Known Issues (Nov 2025)

| Issue                     | Status     | Impact                            |
| ------------------------- | ---------- | --------------------------------- |
| **Unity 6 Shader Format** | 🔄 Partial | 1,082 shaders fail to fully parse |
| TypeTree mismatches       | ✅ Fixed   | Graceful skipping in v2.3.1       |
| Stripped versions         | ✅ Fixed   | Interactive version prompt        |

**Unity 6 Problem**: Format changed without documentation, shaders load with partial data.

---

## 🎨 Logging Levels

```csharp
Logger.Error("...")    // File I/O failures, critical errors
Logger.Warning("...")  // Parse failures (recoverable)
Logger.Info("...")     // User-facing progress
Logger.Verbose("...")  // Debug info (disabled by default)
```

**Always include context:**

```csharp
Logger.Warning($"Failed to parse {m_Name} (Unity {version[0]}.{version[1]}.{version[2]}): {ex.Message}");
```

---

## 🧵 Thread Safety

**AssetStudio is HEAVILY multi-threaded:**

✅ **Do:**

- Use locks for shared collections
- Use `ConcurrentDictionary` for concurrent access
- Keep locks short (minimal hold time)
- Ensure Logger is thread-safe

❌ **Don't:**

- Modify `assetsFileList` without `assetsFileListLock`
- Access shared state without synchronization
- Hold multiple locks (deadlock risk)

---

## 🔍 Debugging Binary Formats

### When encountering parse errors:

1. **Check Unity version** at error location:

   ```csharp
   Logger.Verbose($"Parsing at version {reader.version[0]}.{reader.version[1]}");
   ```

2. **Log stream position**:

   ```csharp
   Logger.Verbose($"Position: {reader.Position}, remaining: {reader.BaseStream.Length - reader.Position}");
   ```

3. **Compare with similar versions** (e.g., 2021.3 vs 2022.1)

4. **Add version-specific path** when format differs

5. **Wrap in try-catch** for graceful degradation

---

## 🛠️ Common Tasks

### Adding Unity Version Support

1. Find affected class (check error logs)
2. Add version check before new fields
3. Test with real game files
4. Update README.md

### Fixing Parse Failures

1. Identify class from error (`Shader.cs`, `Texture2D.cs`, etc.)
2. Check if new fields added in that Unity version
3. Add version-conditional reading
4. Ensure alignment after variable-length data

### Improving Performance

1. Profile with real data (gigabytes)
2. Use `Parallel.ForEach()` for batch operations
3. Preallocate collections (avoid growing)
4. Cache lookups in dictionaries

---

## 📦 Build & Release

```powershell
# Build
dotnet build AssetStudio.sln -c Release

# Test specific project
dotnet build AssetStudio.CLI/AssetStudio.CLI.csproj -c Release

# Release (triggers GitHub Actions)
echo "2.3.2" > VERSION
git add .
git commit -m "v2.3.2 - Description"
git tag v2.3.2
git push origin main --tags
```

---

## 📚 Unity Serialization Types

| C# Type   | Unity Type | Read Method                  |
| --------- | ---------- | ---------------------------- |
| `int`     | Int32      | `reader.ReadInt32()`         |
| `float`   | Single     | `reader.ReadSingle()`        |
| `string`  | String     | `reader.ReadAlignedString()` |
| `bool`    | Boolean    | `reader.ReadBoolean()`       |
| `byte[]`  | UInt8[]    | `reader.ReadUInt8Array()`    |
| `Vector3` | Vector3    | 3x `ReadSingle()`            |
| `PPtr<T>` | Reference  | `new PPtr<T>(reader)`        |

**After variable-length reads (strings, arrays):**

```csharp
reader.AlignStream();  // ⚠️ CRITICAL
```

---

## ⚡ Performance Tips

1. **Preallocate collections:**

   ```csharp
   var items = new List<MyType>(count);  // Not new List<MyType>()
   ```

2. **Use parallel processing:**

   ```csharp
   Parallel.ForEach(collection, item => { /* process */ });
   ```

3. **Cache lookups:**

   ```csharp
   if (!cache.TryGetValue(key, out var value))
   {
       value = ExpensiveOperation();
       cache[key] = value;
   }
   ```

4. **Stream large data** (don't load all into memory)

---

## 🎮 Test Games

| Game                  | Unity Version | Notes                         |
| --------------------- | ------------- | ----------------------------- |
| **Marvel Snap**       | 6000.0.58f2   | Latest format (shader issues) |
| **Genshin Impact**    | 2020.3.x      | MiHoYo encryption             |
| **Among Us**          | 2019.4.x      | Simple baseline               |
| **Honkai: Star Rail** | 2021.3.x      | Good mid-range test           |

---

## 🔗 Related Docs

- **Full Onboarding**: `AI_ONBOARDING.md`
- **User Guide**: `README.md`
- **Release Process**: `RELEASE.md`

---

## 💡 Quick Troubleshooting

| Error                                 | Likely Cause               | Fix                                  |
| ------------------------------------- | -------------------------- | ------------------------------------ |
| "Unable to read beyond end of stream" | Format changed             | Add version check, wrap in try-catch |
| "String length [huge number]"         | Missed field before string | Check for new fields, alignment      |
| Deadlock                              | Lock ordering              | Use consistent lock order            |
| Corrupt exports                       | Wrong deserialization      | Verify field order, types            |

---

## 🎯 Philosophy

**Graceful Degradation Over Crashes**

- Partial data > total failure
- Log warnings, don't throw exceptions
- Users prefer "couldn't export X" over "app crashed"

**Version Agnostic Where Possible**

- Check version only when format differs
- Use try-catch for unknown formats
- Log version info for debugging

**Performance Matters**

- Games have GIGABYTES of assets
- Multi-threading is not optional
- Cache aggressively, allocate smartly

---

**Full docs**: See `AI_ONBOARDING.md` for comprehensive guide.
