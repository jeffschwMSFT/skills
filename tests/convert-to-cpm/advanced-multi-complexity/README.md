# advanced-multi-complexity

A repository with a solution containing five projects that combines multiple CPM conversion complexities in a single scenario. The agent must identify and present all issues together, handle each with separate user interactions, and maintain a coherent conversion plan.

## Setup

```
📂 repo/
├── 📄 Enterprise.slnx
├── 📄 Directory.Build.props
├── 📄 Common.props
├── 📂 Web/
│   └── 📄 Web.csproj
├── 📂 Api/
│   └── 📄 Api.csproj
├── 📂 Core/
│   └── 📄 Core.csproj
├── 📂 Legacy/
│   └── 📄 Legacy.csproj
└── 📂 Tests/
    └── 📄 Tests.csproj
```

`Directory.Build.props`:

```xml
<Project>
  <PropertyGroup>
    <LangVersion>latest</LangVersion>
    <ImplicitUsings>enable</ImplicitUsings>
    <BlobsVersion>12.24.0</BlobsVersion>
    <HostingVersion>8.0.1</HostingVersion>
  </PropertyGroup>
  <Import Project="Common.props" />
</Project>
```

`Common.props`:

```xml
<Project>
  <ItemGroup>
    <PackageReference Include="Microsoft.Extensions.Logging" Version="9.0.0" />
  </ItemGroup>
</Project>
```

`Web/Web.csproj`:

```xml
<Project Sdk="Microsoft.NET.Sdk.Web">
  <PropertyGroup>
    <TargetFramework>net8.0</TargetFramework>
  </PropertyGroup>
  <ItemGroup>
    <PackageReference Include="System.Text.Json" Version="10.0.1" />
    <PackageReference Include="OpenTelemetry.Extensions.Hosting" Version="1.15.0" />
  </ItemGroup>
</Project>
```

`Api/Api.csproj`:

```xml
<Project Sdk="Microsoft.NET.Sdk.Web">
  <PropertyGroup>
    <TargetFrameworks>net8.0;net6.0</TargetFrameworks>
  </PropertyGroup>
  <ItemGroup>
    <PackageReference Include="System.Text.Json" Version="10.0.1" />
    <PackageReference Include="Azure.Storage.Blobs" Version="$(BlobsVersion)" />
  </ItemGroup>
  <ItemGroup Condition="'$(TargetFramework)' == 'net6.0'">
    <PackageReference Include="Microsoft.AspNetCore.Mvc.NewtonsoftJson" Version="6.0.36" />
  </ItemGroup>
  <ItemGroup Condition="'$(TargetFramework)' == 'net8.0'">
    <PackageReference Include="Microsoft.AspNetCore.Mvc.NewtonsoftJson" Version="8.0.11" />
  </ItemGroup>
</Project>
```

`Core/Core.csproj`:

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net8.0</TargetFramework>
  </PropertyGroup>
  <ItemGroup>
    <PackageReference Include="System.Text.Json" Version="9.0.0" />
    <PackageReference Include="Microsoft.Extensions.Hosting" Version="$(HostingVersion)" />
  </ItemGroup>
</Project>
```

`Legacy/Legacy.csproj`:

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net6.0</TargetFramework>
  </PropertyGroup>
  <ItemGroup>
    <PackageReference Include="System.Text.Json" Version="8.0.4" />
    <PackageReference Include="Azure.Storage.Blobs" Version="12.19.0" />
  </ItemGroup>
</Project>
```

`Tests/Tests.csproj`:

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net8.0</TargetFramework>
  </PropertyGroup>
  <ItemGroup>
    <PackageReference Include="xunit" Version="2.9.3" />
    <PackageReference Include="System.Text.Json" Version="10.0.1" />
  </ItemGroup>
</Project>
```

## Complexities present

1. **Version conflicts (System.Text.Json)**: 10.0.1 in Web/Api/Tests, 9.0.0 in Core, 8.0.4 in Legacy — three different major versions, and 8.0.4 has a known security advisory (CVE-2024-43485)
2. **Version conflicts (Azure.Storage.Blobs)**: `$(BlobsVersion)` → 12.24.0 in Api, literal 12.19.0 in Legacy — minor version difference
3. **MSBuild properties**: `$(BlobsVersion)` and `$(HostingVersion)` in `Directory.Build.props`
4. **Conditional PackageReference**: `Microsoft.AspNetCore.Mvc.NewtonsoftJson` uses target framework conditions in Api.csproj with different versions per TFM
5. **Shared .props file**: `Common.props` contains a `PackageReference` for `Microsoft.Extensions.Logging` that applies to all projects
6. **Non-version properties**: `$(LangVersion)` and `$(ImplicitUsings)` in `Directory.Build.props` must not be removed

## Input prompt

I need to convert this entire repository to Central Package Management. The solution is at Enterprise.slnx. There are some complications: package versions defined as MSBuild properties, conditional package references for multi-targeting, and a shared Common.props that adds a package to all projects.

## What the skill should produce

- The agent performs a thorough audit across all 5 projects plus `Common.props` and `Directory.Build.props`
- Presents a comprehensive summary showing all 6 complexities
- Asks the user about each decision point separately:
  - How to resolve the three-way System.Text.Json conflict (including the security advisory on 8.0.4)
  - How to handle the Azure.Storage.Blobs version mismatch (property vs. literal, different versions)
  - Whether to inline or keep each MSBuild property
  - How to handle the conditional PackageReference (conditional `PackageVersion` vs. `VersionOverride`)
  - How to handle the PackageReference in `Common.props`
- Creates `Directory.Packages.props` reflecting all decisions
- Removes only version-related properties, preserving `$(LangVersion)` and `$(ImplicitUsings)`
- Validates with `dotnet restore` and `dotnet build`
- Presents a complete summary noting any items that need ongoing attention
