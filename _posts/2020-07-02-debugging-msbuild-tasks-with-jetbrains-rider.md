---
title: Implementing, Wiring and Debugging Custom MSBuild Tasks
tags:
- MSBuild
- NuGet
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

- [Refit](https://github.com/reactiveui/refit) – REST APIs are generated based on interface declarations [before compilation](https://github.com/reactiveui/refit/blob/master/Refit/targets/refit.targets)
- [Fody](https://github.com/Fody/Fody) – IL code gets rewritten [after compilation](https://github.com/Fody/Fody/blob/master/Fody/Fody.targets) to [add null-checks](https://github.com/Fody/NullGuard), [merge assemblies to a single file](https://github.com/Fody/Costura), [notify on property changes](https://github.com/Fody/PropertyChanged) and [much more](https://github.com/Fody/Home/blob/master/pages/addins.md)
- [NSwag](https://github.com/RicoSuter/NSwag) – Swagger specifications, C#/TypeScript clients and proxies are generated from C# ASP.NET controllers [as part of the build](https://github.com/RicoSuter/NSwag/wiki/NSwag.MSBuild)
- [GitVersion](https://github.com/GitTools/GitVersion) – [Semantic versions](https://semver.org/) are calculated based on our commit history and propagated into MSBuild properties [before compilation](https://github.com/GitTools/GitVersion/blob/master/src/GitVersionTask/build/GitVersionTask.targets) to make them part of the assembly metadata

Some of the scenarios that involve code generation can probably move to [Source Generators](https://devblogs.microsoft.com/dotnet/introducing-c-source-generators/), which are already available in the [.NET 5 previews](https://dotnet.microsoft.com/download/dotnet/5.0). Source generators remove a lot of the burden of writing MSBuild tasks – and sharing [workspaces](https://docs.microsoft.com/en-us/dotnet/csharp/roslyn-sdk/work-with-workspace) – but still aren't easy to debug:

<div class="tweet" tweetID="1258485353070989312">It should be available for debugging, its a pain to debug this thing but much nicer than writing an msbuild task.</div>

Clearly that sets the mood for what a pleasure it is to write an MSBuild task 😅

## MSBuild Integration Options

When we want hook into the execution of MSBuild, we have several options:

- [Inline Tasks](https://docs.microsoft.com/en-us/visualstudio/msbuild/msbuild-roslyncodetaskfactory) – We can write code fragments directly into `.targets` files. They will be compiled into tasks by the `RoslynCodeTaskFactory` and executed by MSBuild when run. This is great for drafting ideas, but falls short in maintainability and debugging.
- [Exec Tasks](https://docs.microsoft.com/visualstudio/msbuild/exec-task) – Any executable can be invoked in a similar way to `Process.Start`. We can capture output, validate exit codes, or define regular expressions for custom error/warning messages. However, we will miss some integration points, and if we need to get complex data _out_ of the process, we'll have to [encode it in a single line](https://github.com/nuke-build/nuke/blob/37503dffe64a1a4a2aac62758dfd4d601e7cb65e/source/Nuke.MSBuildTaskRunner/Program.cs#L163) and [decode it in the target](https://github.com/nuke-build/nuke/blob/37503dffe64a1a4a2aac62758dfd4d601e7cb65e/source/Nuke.MSBuildTaskRunner/Nuke.MSBuildTaskRunner.targets#L42-L54).
- [Custom Tasks](https://docs.microsoft.com/visualstudio/msbuild/task-writing) – We can operate with MSBuild infrastruture directly, including `ITask`, `IBuildEngine`, and `ITaskItem` objects. This allows us to log error/warning/info events, inspect item groups with their [metadata](https://docs.microsoft.com/visualstudio/msbuild/msbuild-well-known-item-metadata), and create more object-like results.

Since **custom tasks are the most scalable solution**, we will put focus on them for the rest of this article.

## Implementing the Task

From my own experience, I'd recommend to **keep the task assembly small** and move complex logic into their own projects. This is also the approach most of the projects mentioned above have taken. Some of the most important types for custom MSBuild tasks are `ITask`, `IBuildEngine`, and `ITaskItem`. In order to gain access, we need to **add references** for them:

{% highlight xml linenos %}
<ItemGroup>
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

Note that the **MSBuild references** are part of every MSBuild installation and therefore **should not be deployed with our final NuGet package**. In order to achieve that, we set the `CopyLocal` and `Publish` attribute for each reference to `false`. We will also need to remove those references from the `ReferenceCopyLocalPaths` item group:

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

As already hinted, our task assembly will likely have dependencies to other projects or NuGet packages. A while back, this would have taken a huge effort:

<div class="tweet" tweetID="882946773332803584">task assemblies that have dependencies on other assemblies is really messy in MSBuild 15. Working around it could be its own blog post</div>

Meanwhile we're at MSBuild 16 already, and some of the [problems that Nate described](https://natemcmaster.com/blog/2017/11/11/msbuild-task-with-dependencies/) in his blog have been addressed. I am by no means an expert in properly resolving dependencies, but [Andrew Arnott](https://twitter.com/aarnott) came up with the `ContextAwareTask` – originally [used in Nerdbank.GitVersion](https://github.com/dotnet/Nerdbank.GitVersioning/blob/3e4e1f8249ba70fd576b524ce12398ee398884fc/src/Nerdbank.GitVersioning.Tasks/ContextAwareTask.cs) – and it's working out great for many folks. As for the actual implementation, we won't go into too much detail but will just look at a basic example:

{% highlight batch linenos %}
using System;
using Microsoft.Build.Framework;
using Microsoft.Build.Utilities;

public class CustomTask : ContextAwareTask
{
    [Required]
    public string Property { get; set; }

    [Required]
    public ITaskItem[] Items { get; set; }

    public override bool Execute()
    {
        return true;
    }
}
{% endhighlight %}

## Wiring the Task

## Creating a NuGet Package

Figuring out how MSBuild works in different environments can take a while. For instance, a project targeting `netcoreapp2.1` would still use MSBuild running on .NET Framework inside Visual Studio, while the same project would be compiled with MSBuild for .NET Core when calling `dotnet build` on the same workstation.

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

On pitfall for me was that loading the MSBuild task is not subject to [NuGet's support multiple .NET versions](https://docs.microsoft.com/en-us/nuget/create-packages/supporting-multiple-target-frameworks), meaning that when we switch our target framework from `net47` to `netcoreapp2.1`, it won't use a different 

## Debugging a Task

Let's get to the most interesting part of how we can **effectively debug the implementation** of our MSBuild task:

<div class="tweet" tweetID="1183638078767927297">Is there a decent workflow for writing MSBuild tasks? Finally looking at adding a proper MSBuild task to FunctionMonkey (it currently uses a console app - that'll stay too). Writing it seems straightforward.... debugging it looks like it might be painful.</div>

Indeed, with the **console application approach**, things are rather easy. <!-- TODO: INPUT DATA --> On Windows we can call `System.Diagnostics.Debugger.Launch()`, which **fires up the Just-In-Time Debugger** so that we can attach to the process. By default, this will use the [Visual Studio JIT Debugger](https://docs.microsoft.com/en-us/visualstudio/debugger/debug-using-the-just-in-time-debugger?view=vs-2019), but we can also [configure JetBrains Rider as the Just-In-Time Debugger](https://blog.jetbrains.com/dotnet/2019/04/16/edit-continue-just-time-debugging-debugger-improvements-rider-2019-1/). As of now, this strategy is [not supported on Linux/macOS](https://github.com/dotnet/runtime/issues/38427). The best workaround I've found is to call `SpinWait.SpinUntil(() => Debugger.IsAttached)`, which will wait until the debugger is actually attached. This also has the benefit that we don't break on the `Debugger.Launch` statement.

Taking the **custom MSBuild task way**, we have a bit more footwork to do. In contrast to using `PackageReference` with the final package, the`.targets` and `.props` files **aren't automatically imported** when using a `ProjectReference`. We could create an actual package, but that has the unpleasant side-effect of getting persisted in our global NuGet cache:

<div class="tweet" tweetID="965325828455321600">I want to point to to a nupckg file directly ideally. The problem with local feeds is that packages get cached.</div>

Deleting from the cache, or incrementing versions – all those are rather poor workarounds compared to a possible [better development and testing experience](https://github.com/NuGet/Home/issues/6579) for NuGet packages. So lets dismiss this idea.

As a first step to enable first-class debugging, we will call `dotnet publish --framework netcoreapp2.1` to **publish the MSBuildTasks project**. When using JetBrains Rider, the most efficient way is to create a [run configuration](https://www.jetbrains.com/help/rider/Run_Debug_Configuration.html):

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