---
title: Implementing and Debugging Custom MSBuild Tasks
tags:
- MSBuild
- .NET
- JetBrains Rider
mode: immersive
header:
  theme: dark
article_header:
  type: overlay
  theme: dark
  background_color: '#203028'
  background_image:
    gradient: 'linear-gradient(135deg, rgba(0, 0, 100 , .5), rgba(60, 34, 60, .4))'
    src: assets/images/2020-02-11-debugging-msbuild-tasks-with-jetbrains-rider/cover.jpg
twitter_card: assets/images/2020-02-11-debugging-msbuild-tasks-with-jetbrains-rider/thumbnail.jpeg
---

<!-- TODO: or even right from the start -->
At some point, much of the tooling around .NET projects ends up having to integrate with MSBuild, the _low-level_ build system in the .NET ecosystem. Examples of these tools are:

- [Refit](https://github.com/reactiveui/refit) – generates type-safe REST APIs based on interface declarations [before compilation](https://github.com/reactiveui/refit/blob/master/Refit/targets/refit.targets)
- [Fody](https://github.com/Fody/Fody) – rewrites IL code [after compilation](https://github.com/Fody/Fody/blob/master/Fody/Fody.targets) and has large variety of addins, including [NullGuard](https://github.com/Fody/NullGuard), [Costura](https://github.com/Fody/Costura), and [PropertyChanged](https://github.com/Fody/PropertyChanged)
- [NSwag](https://github.com/RicoSuter/NSwag) – generates Swagger specifications from C# ASP.NET controllers along with C# and TypeScript clients/proxies [as part of the build](https://github.com/RicoSuter/NSwag/wiki/NSwag.MSBuild)
- [GitVersion](https://github.com/GitTools/GitVersion) – generates [semantic versions](https://semver.org/) based on our commit history, and populates MSBuild properties [before compilation](https://github.com/GitTools/GitVersion/blob/master/src/GitVersionTask/build/GitVersionTask.targets) to make them part of the metadata

Much of the scenarios that involve code generation can eventually move to [Source Generators](https://devblogs.microsoft.com/dotnet/introducing-c-source-generators/), which are already available in the [.NET 5 previews](https://dotnet.microsoft.com/download/dotnet/5.0). Source generators remove a lot of the burden of writing MSBuild tasks – and sharing [workspaces](https://docs.microsoft.com/en-us/dotnet/csharp/roslyn-sdk/work-with-workspace) – but still aren't great with debugging:

<div class="tweet" tweetID="1258485353070989312">It should be available for debugging, its a pain to debug this thing but much nicer than writing an msbuild task.</div>

## Alternative Approaches

Instead of writing full-

I'll mention that you can also write [MSBuild Inline Tasks](https://docs.microsoft.com/en-us/visualstudio/msbuild/msbuild-roslyncodetaskfactory), but I don't consider this a scalable solution, since we can't use any refactoring tools then.

[Use Exec tasks and delegate to console applications](https://github.com/RicoSuter/NSwag/blob/master/src/NSwag.ApiDescription.Client/NSwag.ApiDescription.Client.targets) => Easier integrate and debug, but also misses integration points, like for warnings messages or item group modifications.

Command line => [encoding information in a single line](https://github.com/nuke-build/nuke/blob/37503dffe64a1a4a2aac62758dfd4d601e7cb65e/source/Nuke.MSBuildTaskRunner/Program.cs#L163), and then later [decode it in the MSBuild task](https://github.com/nuke-build/nuke/blob/37503dffe64a1a4a2aac62758dfd4d601e7cb65e/source/Nuke.MSBuildTaskRunner/Nuke.MSBuildTaskRunner.targets#L42-L54)


## Implementing the Task

From my own experience, I'd recommend to keep the task assembly rather small and move complex logic into their own projects. This is also the approach most of the projects mentioned above take.

Some of the most important types for custom MSBuild tasks are `ITask`, `IBuildEngine`, and `ITaskItem`. In order to gain access, we need to add a couple of references:

{% highlight xml linenos %}
<ItemGroup>
  <!-- MSBuild and dependencies only acquired through MSBuild shall not make it into the final package -->
  <PackageReference Include="Microsoft.Build.Utilities.Core" Version="16.3.0" CopyLocal="false" Publish="false" ExcludeAssets="runtime" />
  <PackageReference Include="Microsoft.Build.Framework" Version="16.3.0" CopyLocal="false" Publish="false" ExcludeAssets="runtime" />
  <PackageReference Include="System.Collections.Immutable" Version="1.6.0" CopyLocal="false" Publish="false" />
  <PackageReference Include="System.Runtime.InteropServices.RuntimeInformation" Version="4.3.0" CopyLocal="false" Publish="false" />
</ItemGroup>

<ItemGroup Condition="'$(TargetFrameworkIdentifier)' == '.NETFramework'">
  <PackageReference Include="Microsoft.VisualStudio.Setup.Configuration.Interop" Version="1.16.30" CopyLocal="false" Publish="false" />
</ItemGroup>

<ItemGroup Condition="'$(TargetFrameworkIdentifier)' == '.NETCoreApp'">
  <PackageReference Include="System.Text.Encoding.CodePages" Version="4.6.0" CopyLocal="false" Publish="false" />
</ItemGroup>
{% endhighlight %}

The referenced MSBuild assemblies are part of every 

Note that need to make sure that the referenced MSBuild assemblies are not getting copied into the final NuGet package, because they're . In order to achieve that, we set the `CopyLocal` and `Publish` attribute for each reference to `false`. As a second step, we will remove those references from the `ReferenceCopyLocalPaths` item group:

Cleanup Microsoft assemblies from build output:

{% highlight xml linenos %}
<Target Name="RemoveMicrosoftBuildDllsFromOutput" AfterTargets="ResolveReferences">
  <PropertyGroup>
    <NonCopyLocalPackageReferences Condition="'%(PackageReference.CopyLocal)' == 'false'">;@(PackageReference);</NonCopyLocalPackageReferences>
  </PropertyGroup>
  <ItemGroup>
    <ReferenceCopyLocalPaths Remove="@(ReferenceCopyLocalPaths)" Condition="$(NonCopyLocalPackageReferences.Contains(';%(ReferenceCopyLocalPaths.NuGetPackageId);'))" />
  </ItemGroup>
</Target>
{% endhighlight %}

## Creating the NuGet Package

Now -> [include MSBuild targets](https://docs.microsoft.com/en-us/nuget/create-packages/creating-a-package#include-msbuild-props-and-targets-in-a-package) in the package.
Put custom MSBuild task and targets files into NuGet package:

{% highlight xml linenos %}
<ItemGroup Condition="'$(TargetFramework)' == ''">
  <None Include="$(MSBuildProjectName).props" PackagePath="build" Pack="true" />
  <None Include="$(MSBuildProjectName).targets" PackagePath="build" Pack="true" />
  <None Include="..\Nuke.MSBuildTasks\Nuke.MSBuildTasks.targets" PackagePath="build\netcore" Pack="true" />
  <None Include="..\Nuke.MSBuildTasks\Nuke.MSBuildTasks.targets" PackagePath="build\netfx" Pack="true" />
  <None Include="..\Nuke.MSBuildTasks\bin\$(Configuration)\netcoreapp2.1\publish\**\*.*" PackagePath="build\netcore" Pack="true" />
  <None Include="..\Nuke.MSBuildTasks\bin\$(Configuration)\net472\publish\**\*.*" PackagePath="build\netfx" Pack="true" />
</ItemGroup>
{% endhighlight %}



{% highlight xml linenos %}
<?xml version="1.0" encoding="utf-8"?>
<Project ToolsVersion="4.0" DefaultTargets="Build" xmlns="http://schemas.microsoft.com/developer/msbuild/2003">

  <PropertyGroup Condition="'$(NukeMSBuildTasks)' == ''">
    <NukeMSBuildTasks Condition="'$(MSBuildRuntimeType)' == 'Core'">$(MSBuildThisFileDirectory)\netcore</NukeMSBuildTasks>
    <NukeMSBuildTasks Condition="'$(MSBuildRuntimeType)' != 'Core'">$(MSBuildThisFileDirectory)\netfx</NukeMSBuildTasks>
  </PropertyGroup>

  <Import Project="$(NukeMSBuildTasks)\Nuke.MSBuildTasks.targets" Condition="'$(NukeMSBuildIntegration)' != 'False'" />
</Project>
{% endhighlight %}

On pitfall for me was that loading the MSBuild task is not subject to [NuGet's support multiple .NET versions](https://docs.microsoft.com/en-us/nuget/create-packages/supporting-multiple-target-frameworks), meaning that when we switch our target framework from `net47` to `netcoreapp2.1`, it won't use a diffferent 

## Debugging the Task

Let's get to the most interesting part of how we can **effectively debug the implementation** of our MSBuild task:

<div class="tweet" tweetID="1183638078767927297">Is there a decent workflow for writing MSBuild tasks? Finally looking at adding a proper MSBuild task to FunctionMonkey (it currently uses a console app - that'll stay too). Writing it seems straightforward.... debugging it looks like it might be painful.</div>

Indeed, with the **console application approach**, things are rather easy. <!-- TODO: INPUT DATA --> On Windows we can call `System.Diagnostics.Debugger.Launch()`, which **fires up the Just-In-Time Debugger** so that we can attach to the process. By default, this will use the [Visual Studio JIT Debugger](https://docs.microsoft.com/en-us/visualstudio/debugger/debug-using-the-just-in-time-debugger?view=vs-2019), but we can also [configure Rider as the Just-In-Time Debugger](https://blog.jetbrains.com/dotnet/2019/04/16/edit-continue-just-time-debugging-debugger-improvements-rider-2019-1/). As of now, this strategy is [not supported on Linux/macOS](https://github.com/dotnet/runtime/issues/38427). The best workaround I've found is to call `SpinWait.SpinUntil(() => Debugger.IsAttached)`, which will wait until the debugger is actually attached. This also has the benefit that we don't break on the `Debugger.Launch` statement.

Taking the **custom MSBuild task way**, we have a bit more footwork to do. In contrast to using `PackageReference` with the final package, the **`.targets` and `.props` files aren't automatically imported when using a `ProjectReference`**[^1].

[^1]: At some point in the past I was creating a package for every new version that I've tested. This had the unpleasant side-effects of taking a lot of time for both packing and restore, permanently increasing the version (thus commiting to the repository using GitVersion), and filling my NuGet cache directory with garbage.

As a first step to enable first-class debugging, we will call `dotnet publish --framework netcoreapp2.1` to **publish the MSBuildTasks project**. When using Rider, the most efficient way is to create a [run configuration](https://www.jetbrains.com/help/rider/Run_Debug_Configuration.html):

![Publishing MSBuild Tasks via Run Configuration](/assets/images/2020-02-11-debugging-msbuild-tasks-with-jetbrains-rider/run-configuration-publish.png){:width="750px" .shadow}

In the second step, we call `MSBuild.dll /t:Clean;Restore;Pack /p:NukeMSBuildIntegration=True` to **invoke MSBuild on the project** that we want to test our MSBuild task with:

![Running MSBuild Tasks via Run Configuration](/assets/images/2020-02-11-debugging-msbuild-tasks-with-jetbrains-rider/run-configuration-run.png){:width="750px" .shadow}

![Running MSBuild Tasks via Run Configuration](/assets/images/2020-02-11-debugging-msbuild-tasks-with-jetbrains-rider/run-configuration-list.png){:width="750px" .shadow}

Manually reference props/targets files, as they won't be imported when referencing as ProjectReference. Packing each time is also possible, but cumbersome:

{% highlight xml linenos %}
<Project Sdk="Microsoft.NET.Sdk">

  <Import Project="..\source\Nuke.Common\Nuke.Common.props" />

  <PropertyGroup>
    <OutputType>Exe</OutputType>
    <TargetFramework>netcoreapp3.1</TargetFramework>
    <NukeMSBuildIntegration Condition="'$(NukeMSBuildIntegration)' == ''">False</NukeMSBuildIntegration>
    <NukeMSBuildTasks>$(MSBuildThisFileDirectory)\..\source\Nuke.MSBuildTasks\bin\Debug\netcoreapp2.1\publish</NukeMSBuildTasks>
  </PropertyGroup>

  <ItemGroup>
    <ProjectReference Include="..\source\Nuke.Common\Nuke.Common.csproj" />
  </ItemGroup>

  <Import Project="..\source\Nuke.Common\Nuke.Common.targets" />

</Project>
{% endhighlight %}

## Acknowledgements

I want to add that much of my adventures with MSBuild are only of good nature and happy endings because [Martin Ullrich](https://twitter.com/dasmulli) is such a great source of knowledge:

<div class="tweet" tweetID="1189873542906683392">I think everyone should have a @dasmulli to effectively work with #msbuild. Thank you so much Martin! 👏🏻</div>

If you don't follow him yet, you really should. And sorry to Martin for sending more MSBuild enthusiasts your way! 🤗
