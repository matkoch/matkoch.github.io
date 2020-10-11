# X Things you need to know about MSBuild

In this post I'll try to give a cohencise list of MSBuild concepts that have helped me in the past.

## 1. Properties

A [property](https://docs.microsoft.com/en-us/visualstudio/msbuild/msbuild-properties) is a **name-value pair**. Properties are defined in property groups. Every .NET developer has probably seen the following:

```
<PropertyGroup>
    <OutputType>Exe</OutputType>
    <TargetFramework>netcoreapp3.1</TargetFramework>
    <RootNamespace></RootNamespace>
    <IsPackable>True</IsPackable>
    <NoWarn>CS0649;CS0169</NoWarn>
</PropertyGroup>
```

Property values are strings, but can also be stringly-typed booleans or lists, e.g., `IsPackable` and `NoWarn`. They can be defined to have default values and overwritten in other project files. Often this is done in combination with [conditions]. While doing so, we should obey the [evaluation phase].

MSBuild defines plenty of [reserved and well-known properties](https://docs.microsoft.com/en-us/visualstudio/msbuild/msbuild-reserved-and-well-known-properties). Some of the most useful ones for me were:

| Property                   | Scenario                                                                                                                                                                                                |
|----------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `MSBuildRuntimeType`       | Current MSBuild runtime. The value is either `Full` for the MSBuild CLI (currently _always_ used in Visual Studio), `Core` for the dotnet CLI, or `Mono` for the Mono implementation of MSBuild. |
| `OS`                       | Current operating system. The value is either `Windows_NT` or `Unix`. Other operating systems can be distinguished with [property functions](#11-property-functions) | 
| `MSBuildThisFileDirectory` | Directory of the current project file. This can be used to create references to other files. It does not include the final backslash. |
| `MSBuildThisFileName`      | File name without extension of the current project file. Especially with [NuGet embedded project files] and their [task usages] this allows to avoid hardcoded namespace string. |

By default, all properties in MSBuild are [global properties](https://docs.microsoft.com/en-us/visualstudio/msbuild/msbuild-properties#global-properties) and can be overwritten from the outside with the `-p` switch. For instance, to set the configuration to `Debug`, we can invoke `dotnet build -p:Configuration=Debug`.

We may run in situations, when we want to use a common name for a property, but aren't sure whether it is used in some other place of MSBuild. In that case, we can enable `TreatAsLocalProperty` on the `Project` tag. 

## 2. Items and Metadata

[Items](https://docs.microsoft.com/en-us/visualstudio/msbuild/msbuild-items) are the natural consequence of properties. Some of the most common item groups are `Compile`, `Reference`, `Content`, `EmbeddedResource`

```
<ItemGroup>
  <Compile 
</ItemGroup>
```

The item groups `Compile` and `Resource` became less relevant with the new SDK-based project files.

https://www.cazzulino.com/central-package-versions.html
https://stu.dev/managing-package-versions-centrally/

- Include, Remove, Update

[Daniel Cazzulino](https://twitter.com/kzu) has written a very informative article about [central package versions](https://www.cazzulino.com/central-package-versions.html), which incorporates 


https://docs.microsoft.com/en-us/visualstudio/msbuild/msbuild-items#item-definitions

## 3. Evaluation Phase

This is rather a _should know about existence_ rather than _should know precisely_. Understanding the [evaluation phase](https://docs.microsoft.com/en-us/visualstudio/msbuild/build-process-overview#evaluation-phase) is especially important when you feel that properties or item groups are overwritten or not initialized as expected.

- Evaluate environment variables
- Evaluate imports and properties
- Evaluate item definitions
- Evaluate items
- Evaluate UsingTask elements
- Evaluate targets


## 4. File Types



1. PropertyGroups / ItemGroups / Metadata (predefined)
    
2. Conditions (and or)
https://docs.microsoft.com/en-us/visualstudio/msbuild/comparing-properties-and-items?view=vs-2019
3. Evaluation Steps
    https://docs.microsoft.com/en-us/visualstudio/msbuild/build-process-overview?view=vs-2019#evaluation-phase
4. Batching (where ?)
5. Inbuilt functions / call .NET Code (check OS)
    https://docs.microsoft.com/en-us/visualstudio/msbuild/property-functions?view=vs-2019
    https://github.com/dotnet/samples/blob/227b5de3ef84c7188fd862b69de16720b4942c2f/core/interop/IDynamicInterfaceCastable/src/NativeLib/NativeLib.csproj#L44-L46
6. Import / Directory.Build.props / targets (recursive)
    https://twitter.com/dasmulli/status/1038439238029791233
    https://docs.microsoft.com/en-us/visualstudio/msbuild/customize-your-build?view=vs-2019#use-case-multi-level-merging
    https://docs.microsoft.com/en-us/visualstudio/msbuild/customize-your-build?view=vs-2019#import-order
    <Import Project="$([MSBuild]::GetPathOfFileAbove('Directory.Build.props'))" />
    Could be hard to diagnose
7. NuGet packages (props, targets)
8. Environment variables / -p
    https://docs.microsoft.com/en-us/visualstudio/msbuild/msbuild-properties?view=vs-2019
9. Targets: input/output dependencies
    https://docs.microsoft.com/en-us/visualstudio/msbuild/msbuild-targets?view=vs-2019
    https://stackoverflow.com/questions/2816094/when-should-we-call-a-target-using-dependsontargets-and-calltarget
10. Binlogs and Structured Log Viewer
11. VSWhere
12. Tasks (exec, copy)
    https://docs.microsoft.com/en-us/visualstudio/msbuild/msbuild-task-reference?view=vs-2019
    
13. File Headers

```
<?xml version="1.0" encoding="utf-8"?>
<Project ToolsVersion="4.0" xmlns="http://schemas.microsoft.com/developer/msbuild/2003">
```

https://twitter.com/davidfowl/status/1037025548034166784


after.<Solution>.sln.targets
https://docs.microsoft.com/en-us/visualstudio/msbuild/customize-your-build?view=vs-2019#customize-the-solution-build

.user file
https://docs.microsoft.com/en-us/visualstudio/msbuild/customize-your-build?view=vs-2019#user-file


https://docs.microsoft.com/en-us/nuget/reference/msbuild-targets#restore-properties
https://docs.microsoft.com/en-us/nuget/reference/msbuild-targets#restoring-and-building-with-one-msbuild-command
