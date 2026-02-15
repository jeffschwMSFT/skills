# moderate-version-conflicts

A solution with four projects where two packages appear at different versions across projects. The agent must detect the conflicts, present them to the user with options (align to highest version vs. use `VersionOverride`), and apply the user's choice.

## Setup

```
📂 repo/
├── 📄 Inventory.slnx
├── 📂 Api/
│   └── 📄 Api.csproj
├── 📂 Worker/
│   └── 📄 Worker.csproj
├── 📂 Shared/
│   └── 📄 Shared.csproj
└── 📂 Tests/
    └── 📄 Tests.csproj
```

`Api/Api.csproj`:

```xml
<Project Sdk="Microsoft.NET.Sdk.Web">
  <PropertyGroup>
    <TargetFramework>net8.0</TargetFramework>
  </PropertyGroup>
  <ItemGroup>
    <PackageReference Include="System.Text.Json" Version="10.0.1" />
    <PackageReference Include="Azure.Identity" Version="1.13.2" />
  </ItemGroup>
</Project>
```

`Worker/Worker.csproj`:

```xml
<Project Sdk="Microsoft.NET.Sdk.Worker">
  <PropertyGroup>
    <TargetFramework>net8.0</TargetFramework>
  </PropertyGroup>
  <ItemGroup>
    <PackageReference Include="System.Text.Json" Version="8.0.4" />
    <PackageReference Include="OpenTelemetry.Extensions.Hosting" Version="1.15.0" />
  </ItemGroup>
</Project>
```

`Shared/Shared.csproj`:

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net8.0</TargetFramework>
  </PropertyGroup>
  <ItemGroup>
    <PackageReference Include="System.Text.Json" Version="10.0.1" />
    <PackageReference Include="Azure.Identity" Version="1.10.0" />
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

Key conflicts:

- **System.Text.Json**: 10.0.1 in Api, Shared, Tests vs. 8.0.4 in Worker (8.0.4 has a known security advisory — CVE-2024-43485)
- **Azure.Identity**: 1.13.2 in Api vs. 1.10.0 in Shared (minor version difference, but Shared may depend on older Azure SDK behavior)

## Input prompt

Convert Inventory.slnx to Central Package Management.

## What the skill should produce

- The agent audits all 4 projects and presents both version conflicts clearly
- For each conflict, the agent asks the user whether to align to the higher version or use `VersionOverride` for the project on the lower version
- The agent does not silently pick a version for major-version differences
- After the user responds, the conversion proceeds with the chosen strategy
- `dotnet restore` and `dotnet build` validate the result
