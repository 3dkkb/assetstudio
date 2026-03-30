# Unity 6000 Shader Preview

## Summary

This update improves `Shader` preview support for Unity `6000.x` bundles in `AssetStudio.GUI`.

Current behavior:

- Unity 6000 `Shader` assets no longer fail wholesale during `SerializedPass` parsing.
- The preview panel now shows the outer ShaderLab structure for affected shaders.
- When classic `Program` blobs cannot be decoded, the GUI falls back to TypeTree dump instead of showing only a blank/failed preview.
- `PlayerProgram` metadata is now included in preview output when available.

## What Was Changed

### Source Changes

- `AssetStudio/Classes/Shader.cs`
  - Fixed Unity 6000 `SerializedPass` parsing by skipping legacy pass header fields removed in `6000.x`.
  - Kept Unity 6000 `stageCounts` and `m_PlayerSubPrograms` handling enabled.
  - Preserved safer shader parse diagnostics and end-of-object stream alignment.

- `AssetStudio.Utility/ShaderConverter.cs`
  - Hardened shader conversion so single chunk failures do not blank the entire preview.
  - Added `PlayerProgram` preview output.
  - Restricted raw chunk fallback to legacy-looking shader program containers only.
  - Added limited extraction fallback for unknown chunks that may contain embedded DXBC or SPIR-V payloads.

- `AssetStudio.GUI/MainForm.cs`
  - If `Convert()` returns the standard failure placeholder, the GUI now falls back to `Dump()`.

### Build Changes

- Added local `ForceNet8Only=true` support to allow building on machines that only have .NET 8 SDK installed.
- Produced a packaged GUI build from:

```powershell
dotnet build AssetStudio\AssetStudio.csproj -c Debug -p:ForceNet8Only=true
dotnet build AssetStudio.Utility\AssetStudio.Utility.csproj -c Debug -p:ForceNet8Only=true
dotnet build AssetStudio.GUI\AssetStudio.GUI.csproj -c Debug -p:ForceNet8Only=true -p:OutputPath=bin\Debug\net8.0-windows\
```

## Packaged Build

Included in this push:

- `AssetStudioGUI-net8.0-windows-unity6000-shader-preview.zip`

This ZIP contains the runnable GUI build used for validation.

## Known Limitations

- Unity 6000 outer ShaderLab preview is improved, but some `Program` / `PlayerProgram` bytecode blocks still use a newer chunk format that is not fully decoded yet.
- In those cases, preview may show:
  - outer ShaderLab structure only
  - `PlayerProgram` metadata
  - extracted fallback text if an embedded DXBC or SPIR-V payload is detected

## Validation

Validated manually against a Unity `6000.3.10f1` bundle:

- `Shader` assets now preview structured ShaderLab text instead of failing at parse time.
- Partial Unity 6000 preview recovery confirmed in GUI.
