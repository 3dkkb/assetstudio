# AssetStudio

**Version 2.4.1**

AssetStudio is a Unity asset extraction tool for GUI and CLI workflows. This fork targets practical maintenance for newer Unity versions, including Unity 6000.x.

## Downloads

Get packaged builds from [GitHub Releases](https://github.com/3dkkb/assetstudio/releases).

Current release assets:

- `AssetStudioGUI-net8.0-windows-unity6000-shader-preview.zip`

The package contains:

- `AssetStudio.GUI.exe`
- runtime dependencies required by the GUI build

## Highlights

- Unity 2.x through Unity 6000.x asset support
- Multi-threaded loading and export
- GUI and CLI entry points
- FBX export support
- Texture export support for PNG, JPEG, BMP, and WebP
- Unity 6000 shader preview recovery in GUI

## Unity 6000 Status

This repository now includes partial Unity 6000 shader preview recovery.

What works:

- Unity 6000 `Shader` assets no longer fail wholesale during pass parsing
- GUI preview recovers outer ShaderLab structure for many shaders
- `PlayerProgram` metadata is shown when bytecode cannot yet be decoded
- GUI falls back to TypeTree dump when ShaderLab/HLSL reconstruction fails

Current limitation:

- Some Unity 6000 shader bytecode chunks use a newer container format that is not fully decoded yet, so some `Program` / `PlayerProgram` blocks may still be partial

See [UNITY_6000_SHADER_PREVIEW.md](UNITY_6000_SHADER_PREVIEW.md) for implementation details.

## Build

From the repository root:

```powershell
dotnet restore AssetStudio.sln
dotnet build AssetStudio.sln -c Debug
```

Local .NET 8-only build path used for this release:

```powershell
dotnet build AssetStudio\AssetStudio.csproj -c Debug -p:ForceNet8Only=true
dotnet build AssetStudio.Utility\AssetStudio.Utility.csproj -c Debug -p:ForceNet8Only=true
dotnet build AssetStudio.GUI\AssetStudio.GUI.csproj -c Debug -p:ForceNet8Only=true -p:OutputPath=bin\Debug\net8.0-windows\
```

## Usage

### GUI

Run:

```powershell
AssetStudio.GUI\bin\Debug\net8.0-windows\AssetStudio.GUI.exe
```

### CLI

Basic form:

```powershell
dotnet run --project AssetStudio.CLI -- <input_path> <output_path> [options]
```

## Notes

- `AssetStudio.FBXNative` still requires the native Windows/MSVC toolchain when rebuilt from source.
- The included release ZIP is a packaged GUI build intended for direct use on Windows.

## Credits

- [Perfare/AssetStudio](https://github.com/Perfare/AssetStudio)
- [RazTools/Studio](https://github.com/RazTools/Studio)
- [AssetRipper](https://github.com/AssetRipper/AssetRipper)
- [uTinyRipper](https://github.com/mafaca/UtinyRipper)
