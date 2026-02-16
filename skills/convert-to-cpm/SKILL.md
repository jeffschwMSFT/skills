---
name: convert-to-cpm
description: Convert .NET projects and solutions to use NuGet Central Package Management (CPM). Use when the user wants to centralize package versions into a Directory.Packages.props file, remove Version attributes from PackageReference items, and adopt CPM across a repository, solution, or single project.
---

# Convert to Central Package Management

Migrate .NET projects from per-project package versioning to NuGet Central Package Management (CPM). CPM centralizes all package versions into a single `Directory.Packages.props` file, making version governance and upgrades easier across multi-project repositories.

## When to Use

- The user wants to adopt Central Package Management for a .NET repository, solution, or project
- Package versions are scattered across many `.csproj`, `.fsproj`, or `.vbproj` files and the user wants a single source of truth
- The user mentions `Directory.Packages.props`, CPM, or centralizing NuGet versions

## When Not to Use

- The repository already has CPM fully enabled for all in-scope projects
- The user is working with `packages.config`-based projects (must first migrate to `PackageReference`)
- The user wants to manage versions via a custom MSBuild property file without using CPM

## Inputs

| Input | Required | Description |
|-------|----------|-------------|
| Scope | Yes | A project file, solution file, or directory containing .NET projects to convert |
| Version conflict strategy | No | How to resolve cases where the same package has different versions across projects (default: use highest version) |

## Workflow

### Step 1: Determine scope

- **Single project**: User specifies a `.csproj`, `.fsproj`, or `.vbproj`.
- **Solution**: User specifies a `.sln` or `.slnx`. List projects with `dotnet sln list`.
- **Repository/directory**: No specific file given. Find all project files recursively from the first common ancestor directory of all .NET projects in scope.

If the scope is unclear, ask the user.

### Step 2: Establish baseline build

Before making any changes, verify the scope builds successfully and capture a baseline snapshot using the procedure in [baseline-comparison.md](references/baseline-comparison.md). If the baseline build fails, stop and inform the user — the scope must build cleanly before conversion.

### Step 3: Check for existing CPM

Search for any existing `Directory.Packages.props` in scope or ancestor directories. If CPM is already fully enabled, inform the user and stop. If a `Directory.Packages.props` exists without CPM enabled, ask whether to add the property to the existing file or create a new one.

### Step 4: Audit package references

Extract all `<PackageReference>` items and versions from in-scope project files. Also scan `<Import>` elements to discover shared `.props`/`.targets` files containing package references.

Check for complexities: version conflicts, MSBuild property-based versions, conditional references, security advisories, and existing `VersionOverride` usage. See [audit-complexities.md](references/audit-complexities.md) for the full checklist.

Present audit results to the user before proceeding, including:
- A table of each package, its version(s), and which projects use it
- Any version conflicts, security advisories, or complexities requiring decisions

### Step 5: Create or update Directory.Packages.props

Create the file with `dotnet new packagesprops` (.NET 8+) or manually. Add a `<PackageVersion>` entry for each unique package sorted alphabetically. For conditional versions or `VersionOverride` patterns, see [directory-packages-props.md](references/directory-packages-props.md).

### Step 6: Update project files

Remove the `Version` attribute from every `<PackageReference>` that now has a corresponding `<PackageVersion>`. Also update any shared `.props`/`.targets` files identified in step 4.

- Preserve all other attributes (`PrivateAssets`, `IncludeAssets`, `ExcludeAssets`, `GeneratePathProperty`, `Aliases`)
- Preserve conditional `<ItemGroup>` elements — only remove the `Version` attribute within them
- Retain each file's existing indentation style (spaces vs. tabs, indentation depth) and blank lines — do not reformat or reorganize unchanged lines
- Use `VersionOverride` (with user confirmation) when a project needs a different version than the central one

### Step 7: Handle MSBuild version properties

For `PackageReference` items that used MSBuild properties for versions, determine whether to inline the resolved value or keep the property reference in `Directory.Packages.props`. After validation succeeds in step 8, remove inlined version properties from `Directory.Build.props` or other files, verifying they have no remaining references. See [msbuild-property-handling.md](references/msbuild-property-handling.md) for the decision workflow, import order requirements, and cleanup procedure.

### Step 8: Restore and validate

Run a clean restore and build, producing post-conversion artifacts for comparison using the procedure in [baseline-comparison.md](references/baseline-comparison.md). If errors occur, see [validation-and-errors.md](references/validation-and-errors.md) for NuGet error codes and multi-TFM guidance.

### Step 9: Summary and package list comparison

Compare baseline and post-conversion package lists to produce a per-project version diff. See [baseline-comparison.md](references/baseline-comparison.md) for the comparison procedure and table format. Present changes and unchanged packages in separate tables so the user can verify the conversion.

Also present: number of projects converted, packages centralized, any skipped packages, and MSBuild properties kept or removed. Recommend running `dotnet test`.

Save the full summary (including comparison tables, conversion statistics, and recommendations) as a `convert-to-cpm.md` markdown file alongside the binlog and JSON artifacts. Inform the user that this file can be used as a pull request description. Also inform the user that binlog files and package list JSON files are available for manual inspection.

## Validation

- [ ] Baseline build succeeded before any changes were made
- [ ] `Directory.Packages.props` exists with `ManagePackageVersionsCentrally` set to `true`
- [ ] Every in-scope `PackageReference` either has no `Version` attribute or uses `VersionOverride`
- [ ] Every referenced package has a corresponding `PackageVersion` entry
- [ ] `dotnet restore` and `dotnet build` complete without errors from a clean state
- [ ] Package list comparison shows no unexpected version changes
- [ ] No orphaned version properties remain (unless intentionally kept)

## More Info

- [Central Package Management documentation](https://learn.microsoft.com/nuget/consume-packages/central-package-management)
- [Validation and common errors](references/validation-and-errors.md)
