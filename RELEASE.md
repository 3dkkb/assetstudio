# Release Instructions

## How to Create a New Release

This repository uses GitHub Actions to automatically build and publish releases.

### Steps:

1. **Update VERSION file** (if you want to track version in the repo):

   ```bash
   echo "2.0.0" > VERSION
   ```

2. **Commit your changes**:

   ```bash
   git add .
   git commit -m "Release v2.0.0"
   ```

3. **Create and push a version tag**:

   ```bash
   git tag v2.0.0
   git push origin v2.0.0
   ```

4. **Automated build will trigger**:

   - GitHub Actions will automatically build the solution
   - Creates two ZIP files:
     - `AssetStudioCLI-net10.0-win.zip`
     - `AssetStudioGUI-net10.0-win.zip`
   - Publishes them as a GitHub Release

5. **Release will appear** at: `https://github.com/Razviar/assetstudio/releases`

### Version Numbering

Follow [Semantic Versioning](https://semver.org/):

- **Major** (2.x.x): Breaking changes
- **Minor** (x.1.x): New features, backward compatible
- **Patch** (x.x.1): Bug fixes

Examples:

- `v2.0.0` - Initial release with Unity 6 support + .NET 10
- `v2.1.0` - Add new export features
- `v2.0.1` - Fix texture decoding bug

### Manual Release (if needed)

If you need to create a release manually:

1. Build the solution:

   ```powershell
   dotnet build AssetStudio.sln -c Release
   ```

2. Package the files:

   - CLI: `AssetStudio.GUI/bin/Release/net10.0-windows/AssetStudio.CLI.*`
   - GUI: `AssetStudio.GUI/bin/Release/net10.0-windows/*` (entire folder)

3. Create release on GitHub with the ZIP files attached

### First Release

To create your first v2.0.0 release:

```bash
git add .
git commit -m "Release v2.0.0 - Unity 6 support, .NET 10 upgrade"
git tag v2.0.0
git push origin main
git push origin v2.0.0
```

The GitHub Action will handle the rest!
