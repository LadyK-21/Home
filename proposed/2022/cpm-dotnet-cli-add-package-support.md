# Adding Dotnet CLI add package command support to projects onboarded with Central Package Management

- Author: [Pragnya Pandrate](https://github.com/pragnya17), [Kartheek Penagamuri](https://github.com/kartheekp-ms)
- Issue: [11807](https://github.com/NuGet/Home/issues/11807)
- Status: In Review

## Problem Background

The dotnet add package command allows users to add or update a package reference in a project file through the Dotnet CLI. However, when this command is used in a project that has been onboarded to Central Package Management (CPM), it poses an issue as this error is thrown: `error: NU1008: Projects that use central package version management should not define the version on the PackageReference items but on the PackageVersion items: [PackageName]`.

Projects onboarded to CPM use a `Directory.packages.props` file in the root of the repo where package versions are defined centrally. Ideally, when the `dotnet add package` command is used, the package version should only be added to the corresponding package in the `Directory.packages.props` file. However, currently the command attempts to add the package version to the `<PackageReference />` in the project which conflicts with the CPM requirements that package versions must only be in the `Directory.packages.props` file.

## Goals

The main goal is to add support for `dotnet add package` to be used with projects onboarded onto CPM. Regardless of whether the package has already been added to the project or not, the command should allow users to add packages or update the package version in the `Directory.packages.props` file.

## Customers

Users wanting to use CPM onboarded projects and dotnet CLI commands.

## Solution

When `dotnet add package` is executed in a project onboarded to CPM (meaning that the `Directory.packages.props` file exists) there are a few scenarios that must be considered.

| Scenario # | PackageReference exists? | Directory.Packages.Props file exists? | Is ManagePackageVersionsCentrally property set to true? | Is VersionOverride? | Current behavior | New behavior in dotnet CLI | In Scope |
| ---- |-----| -----|-----|----------|----------|----------|----------|
| 1 | ❌ | ✔️ | ✔️ (in Directory.Packages.Props or .csproj file) | ❌ | Restore failed with NU1008 error and NO edits were made to the csproj file (same in VS and dotnet CLI) | `PackageReference` should be added to .(cs/vb)proj file and `PackageVersion` should be added to the closest `Directory.Packages.Props` file | ✔️ |
| 2 | ❌ | ❌ | ✔️ in (cs/vb)proj file | ❌ | Restore failed with NU1008 error and NO edits were made to the csproj file (same in VS and dotnet CLI) | `PackageReference` and `Version` should be added to .(cs/vb)proj file.| ✔️ |
| 3 | ❌ | ✔️ | ❌ | ❌ | `PackageReference` and `Version` added to .(cs/vb)proj file. | In addition to the current behavior `Directory.packages.props` file should be deleted if it exists in the project folder | ❌ |
| 4 | ✔️ | ✔️ | ✔️ | ❌ | **dotnet CLI** - Restore failed with NU1008 error and NO edits were made to the csproj file. **VS** - Clicked on Update package in PM UI. New `PackageVersion` added to csproj file and `Directory.packages.props` file was not updated | No changes should be made to the `Directory.Packages.Props` file. Add `VersionOverride` attribute to the existing `PackageReference` item in .(cs/vb)proj file. If `VersionOverride` is already specified then the value should be updated.|✔️ [More Info](https://github.com/NuGet/Home/pull/11849#discussion_r890639808) |
| 5 | ✔️ | ✔️ | ✔️ | ✔️ | **dotnet CLI** - Restore failed with NU1008 error and NO edits were made to the csproj file. **VS** - Clicked on Update package in PM UI. New `PackageVersion` added to csproj file and `VersionOverride` in .csproj file was not updated | Update `VersionOverride` attribute value in the corresponding `PackageReference` item in .(cs/vb)proj file. | ✔️ |

> [!NOTE]
> Scenarios with multiple Directory.Packages.props are out of scope for now.

### 1. The package reference does not exist

If the package does not already exist, it should be added along with the appropriate package version to `Directory.packages.props`. The package version should either be the latest version or the one specified in the CLI command. Only the package name (not the version) should be added to `<PackageReference>` in the project file.

#### Before `add package` is executed

The props file:

```xml
<Project>
    <PropertyGroup>
        <ManagePackageVersionsCentrally>true</ManagePackageVersionsCentrally>
    </PropertyGroup>
    <ItemGroup>
    </ItemGroup>
</Project>
```

The .csproj file:

```xml
<Project Sdk="Microsoft.NET.Sdk">
    <PropertyGroup>
        <TargetFramework>net6.0</TargetFramework>
        <ImplicitUsings>enable</ImplicitUsings>
        <Nullable>enable</Nullable>
    </PropertyGroup>

    <ItemGroup>
    </ItemGroup>
</Project>
```

#### After `add package` is executed

`dotnet add ToDo.csproj package Newtonsoft.Json -v 13.0.1`

The props file:

```xml
<Project>
    <PropertyGroup>
        <ManagePackageVersionsCentrally>true</ManagePackageVersionsCentrally>
    </PropertyGroup>
    <ItemGroup>
    <PackageVersion Include="Newtonsoft.Json" Version="13.0.1"/>
    </ItemGroup>
</Project>
```

The .csproj file:

```xml
<Project Sdk="Microsoft.NET.Sdk">
    <PropertyGroup>
        <TargetFramework>net6.0</TargetFramework>
        <ImplicitUsings>enable</ImplicitUsings>
        <Nullable>enable</Nullable>
    </PropertyGroup>

    <ItemGroup>
        <PackageReference Include="Newtonsoft.Json"/>
    </ItemGroup>
</Project>
```

In case there are multiple `Directory.packages.props` files in the repo, the props file that is closest must be considered.

```xml
Repository
 |-- Directory.Packages.props
 |-- Solution1
    |-- Directory.Packages.props
    |-- Project1
 |-- Solution2
    |-- Project2
```

In the above example, the following scenarios are possible:

1. Project1 will evaluate the Directory.Packages.props file in the Repository\Solution1\ directory.
2. Project2 will evaluate the Directory.Packages.props file in the Repository\ directory.

***Sourced from <https://devblogs.microsoft.com/nuget/introducing-central-package-management/>

### 2. The package reference does exist

If the package already exists in `Directory.packages.props` the version should be updated in `Directory.packages.props`. The package version should either be the latest version or the one specified in the CLI command. The `<PackageReference>` in the project file should not change.

#### Before `add package` is executed
The props file:

```xml
<Project>
    <PropertyGroup>
        <ManagePackageVersionsCentrally>true</ManagePackageVersionsCentrally>
    </PropertyGroup>
    <ItemGroup>
    <PackageVersion Include="Newtonsoft.Json" Version="12.0.1"/>
    </ItemGroup>
</Project>
```

The .csproj file:

```xml
<Project Sdk="Microsoft.NET.Sdk">
    <PropertyGroup>
        <TargetFramework>net6.0</TargetFramework>
        <ImplicitUsings>enable</ImplicitUsings>
        <Nullable>enable</Nullable>
    </PropertyGroup>

    <ItemGroup>
        <PackageReference Include="Newtonsoft.Json"/>
    </ItemGroup>
</Project>
```

#### After `add package` is executed

`dotnet add ToDo.csproj package Newtonsoft.Json -v 13.0.1`

No changes should be made to the `Directory.Packages.Props` file. Add `VersionOverride` attribute to the existing `PackageReference` item in .(cs/vb)proj file. If `VersionOverride` is already specified then the value should be updated.

The .csproj file:

```xml
<Project Sdk="Microsoft.NET.Sdk">
    <PropertyGroup>
        <TargetFramework>net6.0</TargetFramework>
        <ImplicitUsings>enable</ImplicitUsings>
        <Nullable>enable</Nullable>
    </PropertyGroup>

    <ItemGroup>
        <PackageReference Include="Newtonsoft.Json" VersionOverride="13.0.1"/>
    </ItemGroup>
</Project>
```

### 3. The package reference does exist with an VersionOverride

If the package already exists in `Directory.packages.props` the version should be updated in `Directory.packages.props`. The package version should either be the latest version or the one specified in the CLI command. The `<PackageReference>` in the project file should not change.

#### Before `add package` is executed

```xml
<Project>
    <PropertyGroup>
        <ManagePackageVersionsCentrally>true</ManagePackageVersionsCentrally>
    </PropertyGroup>
    <ItemGroup>
    <PackageVersion Include="Newtonsoft.Json" Version="11.0.1"/>
    </ItemGroup>
</Project>
```

```xml
<Project>
    <PropertyGroup>
        <ManagePackageVersionsCentrally>true</ManagePackageVersionsCentrally>
    </PropertyGroup>
    <ItemGroup>
    <PackageVersion Include="Newtonsoft.Json" VersionOverride="12.0.1"/>
    </ItemGroup>
</Project>
```

#### After `add package` is executed

- No changes should be made to the `Directory.packages.props` file. 
- The version specified in the `VersionOverride` atribute value of `PackageReference` element in the project file should be updated.

`dotnet add ToDo.csproj package Newtonsoft.Json -v 13.0.1`

```xml
<Project>
    <PropertyGroup>
        <ManagePackageVersionsCentrally>true</ManagePackageVersionsCentrally>
    </PropertyGroup>
    <ItemGroup>
    <PackageVersion Include="Newtonsoft.Json" VersionOverride="13.0.1"/>
    </ItemGroup>
</Project>
```
