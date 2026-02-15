# Binlog Comparison

MSBuild binary logs (binlogs) capture the full structured build event stream, including every resolved package reference and version per project. Use binlogs to produce a verifiable before/after comparison of the CPM conversion.

## Producing binlogs

Use the `-bl` flag with a named output file. Always build from a clean state to ensure the log reflects a full resolution, not incremental cache.

### Baseline (before conversion)

```bash
dotnet clean
dotnet build -bl:baseline.binlog
```

### Post-conversion (after all changes)

```bash
dotnet clean
dotnet build -bl:after-cpm.binlog
```

For solution-scoped conversions, pass the solution file:

```bash
dotnet clean MySolution.sln
dotnet build MySolution.sln -bl:baseline.binlog
```

## Comparing package versions

Use `dotnet list package` to capture the resolved package versions before and after conversion. This provides a concrete, tool-based comparison without requiring binlog parsing libraries.

### Capture baseline (before conversion, after step 2 baseline build)

```bash
dotnet list package --format json > baseline-packages.json
```

If `--format json` is not available (requires .NET 8 SDK+), use the default tabular output:

```bash
dotnet list package > baseline-packages.txt
```

For solution-scoped conversions:

```bash
dotnet list MySolution.sln package --format json > baseline-packages.json
```

### Capture post-conversion (after step 8 clean build)

```bash
dotnet list package --format json > after-cpm-packages.json
```

### Produce the comparison

Compare the baseline and post-conversion package lists per project. For each project, identify:

1. **Version changes**: Packages whose resolved version differs.
2. **Added packages**: Packages present after conversion but not in the baseline.
3. **Removed packages**: Packages present in the baseline but not after conversion.
4. **VersionOverride entries**: Packages that use `VersionOverride` (their version matches baseline but the mechanism changed).
5. **Transitive changes**: If `CentralPackageTransitivePinningEnabled` was set, note any transitive packages that are now pinned.

### Changes table

Present all packages with differences first:

```
| Project        | Package                  | Baseline  | After CPM | Change              |
|----------------|--------------------------|-----------|-----------|---------------------|
| Legacy.csproj  | System.Text.Json         | 8.0.4     | 8.0.5     | ⬆️ Security fix      |
| Core.csproj    | System.Text.Json         | 9.0.0     | 9.0.0     | VersionOverride     |
| Shared.csproj  | AutoMapper               | 12.0.1    | 12.0.1    | VersionOverride     |
| Api.csproj     | SomePackage              | 2.0.0     | —         | ❌ Removed           |
| Api.csproj     | NewPackage               | —         | 1.0.0     | ➕ Added             |
```

### Unchanged packages table

Then present all packages that resolved to the same version, confirming version-neutral conversion:

```
| Project        | Package                          | Version   |
|----------------|----------------------------------|-----------|
| Api.csproj     | System.Text.Json                 | 10.0.1    |
| Api.csproj     | Microsoft.EntityFrameworkCore    | 8.0.11    |
| Web.csproj     | System.Text.Json                 | 10.0.1    |
| Web.csproj     | Serilog.AspNetCore               | 8.0.3     |
| Tests.csproj   | xunit                            | 2.9.3     |
| Tests.csproj   | System.Text.Json                 | 10.0.1    |
```

If there are no changes at all, state that the conversion is fully version-neutral.

## Artifacts for the user

After comparison, inform the user that the binlog files are available for manual review:

- `baseline.binlog` — Build state before CPM conversion
- `after-cpm.binlog` — Build state after CPM conversion

These can be opened in the [MSBuild Structured Log Viewer](https://msbuildlog.com/) for detailed inspection of the full build tree, including target execution, property evaluation, and item resolution.

If the user's environment does not have the viewer installed, it can be obtained via:

```bash
# Windows
winget install KirillOsenkov.MSBuildStructuredLogViewer

# Or download from https://msbuildlog.com/
```

## When comparison reveals unexpected differences

If the post-conversion build resolves different package versions than expected (beyond intentional changes like security bumps or `VersionOverride`), investigate:

- Missing `<PackageVersion>` entries causing fallback behavior
- Conditional `<PackageVersion>` entries not matching the project's target framework
- Import order issues where a property referenced in `Directory.Packages.props` is not yet defined
- Transitive dependency resolution differences from version alignment
- Packages unexpectedly added or removed due to conditional ItemGroup changes

Flag any unexpected differences to the user before considering the conversion complete.
