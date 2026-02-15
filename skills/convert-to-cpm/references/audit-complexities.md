# Audit Complexities

When auditing `PackageReference` items across in-scope project files, watch for these complexities and flag them to the user:

## 1. Version set via MSBuild property

If a `PackageReference` uses a property for its version (e.g., `Version="$(SomePackageVersion)"`), trace the property definition. If the property is defined in a `Directory.Build.props`, `.props` import, or the project file itself, note it for the user. These require manual decisions about whether to replace the property with a literal version in `Directory.Packages.props` or to keep the property and use it within `Directory.Packages.props`.

See [msbuild-property-handling.md](msbuild-property-handling.md) for decision workflow.

## 2. Conditional PackageReference items

If a `PackageReference` is inside a conditional `<ItemGroup>` (e.g., `Condition="'$(TargetFramework)' == 'net8.0'"`), the version must still be centralized. The `PackageVersion` entry in `Directory.Packages.props` can use the same condition, or the project can use `VersionOverride` if the condition is project-specific.

## 3. Same package with different versions

If the same package ID appears with different versions across projects, record all versions. The default strategy is to use the highest version.

- **Major version difference**: Ask the user to confirm — may indicate intentional pinning.
- **Minor or patch difference**: Prefer the highest version but note the change — a patch-level difference may indicate a security fix.

## 4. Known security advisories

If a package version is known to have security vulnerabilities (e.g., from nuget.org advisory data or `dotnet list package --vulnerable` output), flag the vulnerable version and recommend upgrading at least to the minimum patched version. Do not silently keep a vulnerable version even if a project pins to it.

## 5. Packages without a Version attribute

These may already be managed by CPM from a parent directory or may be using a default version. Verify whether a `Directory.Packages.props` in an ancestor directory already provides the version.

## 6. PackageReference in imported .props/.targets files

Scan for `<Import>` elements in project files and `Directory.Build.props` to discover shared `.props` or `.targets` files that may contain `PackageReference` items. Search those imported files for package references — they need the same treatment but modifying shared build files has broader impact. Flag these to the user.

## 7. VersionOverride already in use

If any project already uses `VersionOverride`, note it — this suggests partial CPM adoption may already be in progress.
