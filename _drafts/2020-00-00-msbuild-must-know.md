# X Things you need to know about MSBuild

1. PropertyGroups / ItemGroups / Metadata (predefined)
    https://docs.microsoft.com/en-us/visualstudio/msbuild/msbuild-reserved-and-well-known-properties?view=vs-2019
2. Conditions (and or)
3. Evaluation Steps
    https://docs.microsoft.com/en-us/visualstudio/msbuild/build-process-overview?view=vs-2019#evaluation-phase
4. Batching (where ?)
5. Inbuilt functions / call .NET Code (check OS)
    https://docs.microsoft.com/en-us/visualstudio/msbuild/property-functions?view=vs-2019
    https://github.com/dotnet/samples/blob/227b5de3ef84c7188fd862b69de16720b4942c2f/core/interop/IDynamicInterfaceCastable/src/NativeLib/NativeLib.csproj#L44-L46
6. Import / Directory.Build.props / targets (recursive)
    https://twitter.com/dasmulli/status/1038439238029791233
    https://docs.microsoft.com/en-us/visualstudio/msbuild/customize-your-build?view=vs-2019#use-case-multi-level-merging
7. NuGet packages (props, targets)
8. Environment variables / -p
    https://docs.microsoft.com/en-us/visualstudio/msbuild/msbuild-properties?view=vs-2019
9. Targets: input/output dependencies
    https://stackoverflow.com/questions/2816094/when-should-we-call-a-target-using-dependsontargets-and-calltarget
10. Binlogs and Structured Log Viewer
11. VSWhere
12. Tasks (exec, copy)
    https://docs.microsoft.com/en-us/visualstudio/msbuild/msbuild-task-reference?view=vs-2019

https://twitter.com/davidfowl/status/1037025548034166784