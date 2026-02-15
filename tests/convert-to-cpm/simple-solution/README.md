# simple-solution

A .NET solution with three projects. All projects use the same versions for shared packages. No version conflicts, no MSBuild properties, no conditions. The agent should enumerate projects via `dotnet sln list`, create `Directory.Packages.props` at the solution root, and update all three project files.

## Setup

```
📂 repo/
├── 📄 MyApp.sln
├── 📂 Web/
│   └── 📄 Web.csproj
├── 📂 Core/
│   └── 📄 Core.csproj
└── 📂 Tests/
    └── 📄 Tests.csproj
```

`Web/Web.csproj`:

```xml
<Project Sdk="Microsoft.NET.Sdk.Web">
  <PropertyGroup>
    <TargetFramework>net8.0</TargetFramework>
  </PropertyGroup>
  <ItemGroup>
    <PackageReference Include="OpenTelemetry.Extensions.Hosting" Version="1.15.0" />
    <PackageReference Include="Microsoft.AspNetCore.OpenApi" Version="8.0.24" />
  </ItemGroup>
  <ItemGroup>
    <ProjectReference Include="..\Core\Core.csproj" />
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
    <PackageReference Include="Microsoft.NET.Test.Sdk" Version="17.12.0" />
    <PackageReference Include="xunit" Version="2.9.3" />
    <PackageReference Include="xunit.runner.visualstudio" Version="2.8.2" PrivateAssets="all" />
  </ItemGroup>
  <ItemGroup>
    <ProjectReference Include="..\Core\Core.csproj" />
  </ItemGroup>
</Project>
```

## Input prompt

I'd like to convert MyApp.sln to Central Package Management.

## What the skill should produce

- The agent runs `dotnet sln list` to enumerate projects
- Checks for existing `Directory.Packages.props`
- Presents an audit showing 3 projects and 6 unique packages with no conflicts
- Creates `Directory.Packages.props` at the solution root
- Removes `Version` attributes from all three project files, preserving `PrivateAssets` on xunit.runner.visualstudio
- Runs `dotnet restore` and `dotnet build` to validate
