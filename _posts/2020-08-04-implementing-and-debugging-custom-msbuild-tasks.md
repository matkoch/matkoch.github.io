---
title: Implementing and Debugging Custom MSBuild Tasks
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
    src: assets/images/2020-08-04-implementing-and-debugging-custom-msbuild-tasks/cover.jpg
twitter_card: assets/images/2020-08-04-implementing-and-debugging-custom-msbuild-tasks/thumbnail.jpeg
---

<script>
{%- include scripts/lib/swiper.js -%}
var SOURCES = window.TEXT_VARIABLES.sources;
window.Lazyload.js(SOURCES.jquery, function() {
  $('.swiper-demo').swiper();
});
</script>

<style>
  .swiper-demo {
    margin: 30px auto;
  }
  .swiper-demo .swiper__slide {
    display: flex;
    align-items: center;
    justify-content: center;
    font-size: 3rem;
    color: #fff;
  }
  .swiper-demo .swiper__slide:nth-child(even) {
    background-color: #ff69b4;
  }
  .swiper-demo .swiper__slide:nth-child(odd) {
    background-color: #2593fc;
  }
  .swiper-demo--dark .swiper__slide:nth-child(even) {
    background-color: #312;
  }
  .swiper-demo--dark .swiper__slide:nth-child(odd) {
    background-color: #123;
  }
  .swiper-demo--image .swiper__slide:nth-child(n) {
    background-color: #000;
  }
</style>

***TL;DR: Custom MSBuild tasks can be hard at first. With a few tricks and tools the process of implementing, packaging and debugging the infrastructure can be greatly simplified. A sample project is [available on GitHub](https://github.com/matkoch/custom-msbuild-task).***

Much of the tooling around .NET projects ends up having to integrate with [MSBuild](https://docs.microsoft.com/visualstudio/msbuild/msbuild), the _low-level_ build system in the .NET ecosystem. A few examples of these tools are:

- [Refit](https://github.com/reactiveui/refit) – REST APIs are generated based on interface declarations [before compilation](https://github.com/reactiveui/refit/blob/master/Refit/targets/refit.targets)
- [Fody](https://github.com/Fody/Fody) – IL code gets rewritten [after compilation](https://github.com/Fody/Fody/blob/master/Fody/Fody.targets) to [add null-checks](https://github.com/Fody/NullGuard), [merge assemblies to a single file](https://github.com/Fody/Costura), [notify on property changes](https://github.com/Fody/PropertyChanged) and [much more](https://github.com/Fody/Home/blob/master/pages/addins.md)
- [NSwag](https://github.com/RicoSuter/NSwag) – Swagger specifications, C#/TypeScript clients and proxies are generated from C# ASP.NET controllers [as part of the build](https://github.com/RicoSuter/NSwag/wiki/NSwag.MSBuild)
- [GitVersion](https://github.com/GitTools/GitVersion) – [Semantic versions](https://semver.org/) are calculated based on our commit history and propagated into MSBuild properties [before compilation](https://github.com/GitTools/GitVersion/blob/master/src/GitVersionTask/build/GitVersionTask.targets) to make them part of the assembly metadata

Some of the scenarios that involve code generation could eventually move to [Source Generators](https://devblogs.microsoft.com/dotnet/introducing-c-source-generators/), which are already available in the [.NET 5 previews](https://dotnet.microsoft.com/download/dotnet/5.0). Source generators remove a lot of the burden of writing MSBuild tasks – and sharing [workspaces](https://docs.microsoft.com/en-us/dotnet/csharp/roslyn-sdk/work-with-workspace) – but still aren't easy to debug:

<div class="tweet" tweetID="1258485353070989312">It should be available for debugging, its a pain to debug this thing but much nicer than writing an msbuild task.</div>

Clearly that sets the mood for what a pleasure it is to write an MSBuild task! 😅

## MSBuild Integration Options

When we want hook into the execution of MSBuild, we have several options:

- [Inline Tasks](https://docs.microsoft.com/en-us/visualstudio/msbuild/msbuild-roslyncodetaskfactory) – We can write code fragments directly into `.targets` files. They will be compiled into tasks by the `RoslynCodeTaskFactory` and executed by MSBuild when run. This is great for drafting ideas, but falls short in maintainability and debugging.
- [Exec Tasks](https://docs.microsoft.com/visualstudio/msbuild/exec-task) – Any executable can be invoked in a similar way to `Process.Start`. We can capture output, validate exit codes, or define regular expressions for custom error/warning messages. However, we will miss some integration points, and if we need to get complex data _out_ of the process, we'll have to [encode it in a single line](https://github.com/nuke-build/nuke/blob/37503dffe64a1a4a2aac62758dfd4d601e7cb65e/source/Nuke.MSBuildTaskRunner/Program.cs#L163) and [decode it in the target](https://github.com/nuke-build/nuke/blob/37503dffe64a1a4a2aac62758dfd4d601e7cb65e/source/Nuke.MSBuildTaskRunner/Nuke.MSBuildTaskRunner.targets#L42-L54).
- [Custom Tasks](https://docs.microsoft.com/visualstudio/msbuild/task-writing) – We can operate with MSBuild infrastructure directly, including `ITask`, `IBuildEngine`, and `ITaskItem` objects. This allows us to log error/warning/info events, inspect item groups with their [metadata](https://docs.microsoft.com/visualstudio/msbuild/msbuild-well-known-item-metadata), create more object-like results, and even to support [incremental builds](https://docs.microsoft.com/visualstudio/msbuild/incremental-builds).

Since **custom tasks are the most scalable solution**, we will put focus on them for the rest of this article.

## Implementing the Task

From my own experience, I'd recommend to **keep the task assembly small** and move complex logic into their own projects. This is also the approach most of the projects mentioned above have taken. Some of the most important types for custom MSBuild tasks are `ITask`, `IBuildEngine`, and `ITaskItem`. In order to gain access, we need to **add references** for them:

{% highlight xml linenos %}
<ItemGroup>
  <PackageReference Include="Microsoft.Build.Utilities.Core" Version="16.3.0" CopyLocal="false" Publish="false" ExcludeAssets="runtime"/>
  <PackageReference Include="Microsoft.Build.Framework" Version="16.3.0" CopyLocal="false" Publish="false" ExcludeAssets="runtime"/>
  <PackageReference Include="System.Collections.Immutable" Version="1.6.0" CopyLocal="false" Publish="false"/>
  <PackageReference Include="System.Runtime.InteropServices.RuntimeInformation" Version="4.3.0" CopyLocal="false" Publish="false"/>
</ItemGroup>

<ItemGroup Condition="'$(TargetFrameworkIdentifier)' == '.NETFramework'">
  <PackageReference Include="Microsoft.VisualStudio.Setup.Configuration.Interop" Version="1.16.30" CopyLocal="false" Publish="false"/>
</ItemGroup>

<ItemGroup Condition="'$(TargetFrameworkIdentifier)' == '.NETCoreApp'">
  <PackageReference Include="System.Text.Encoding.CodePages" Version="4.6.0" CopyLocal="false" Publish="false"/>
</ItemGroup>
{% endhighlight %}

Note that the **MSBuild references** are part of every MSBuild installation and therefore **should not be deployed with our final NuGet package**. In order to achieve that, we set the `CopyLocal` and `Publish` attribute for each reference to `false`. We will also need to remove those references from the `ReferenceCopyLocalPaths` item group:

{% highlight xml linenos %}
<Target Name="RemoveMicrosoftBuildDllsFromOutput" AfterTargets="ResolveReferences">
  <PropertyGroup>
    <NonCopyLocalPackageReferences Condition="'%(PackageReference.CopyLocal)' == 'false'">;@(PackageReference);</NonCopyLocalPackageReferences>
  </PropertyGroup>
  <ItemGroup>
    <ReferenceCopyLocalPaths Remove="@(ReferenceCopyLocalPaths)" Condition="$(NonCopyLocalPackageReferences.Contains(';%(ReferenceCopyLocalPaths.NuGetPackageId);'))"/>
  </ItemGroup>
</Target>
{% endhighlight %}

As already hinted, our task assembly will likely have dependencies to other projects or NuGet packages. A while back, this would have taken a huge effort:

<div class="tweet" tweetID="882946773332803584">task assemblies that have dependencies on other assemblies is really messy in MSBuild 15. Working around it could be its own blog post</div>

Meanwhile we're at MSBuild 16, and some of the [problems that Nate described](https://natemcmaster.com/blog/2017/11/11/msbuild-task-with-dependencies/) in his blog have already been addressed. I am by no means an expert in properly resolving dependencies, but [Andrew Arnott](https://twitter.com/aarnott) came up with the `ContextAwareTask` – originally [used in Nerdbank.GitVersion](https://github.com/dotnet/Nerdbank.GitVersioning/blob/3e4e1f8249ba70fd576b524ce12398ee398884fc/src/Nerdbank.GitVersioning.Tasks/ContextAwareTask.cs) – which is working out great for many folks:

{% highlight batch linenos %}
using System;
using Microsoft.Build.Framework;
using Microsoft.Build.Utilities;

namespace CustomTasks
{
    public class CustomTask : ContextAwareTask
    {
        [Required] public string StringParameter { get; set; }

        public ITaskItem[] FilesParameter { get; set; }

        protected override bool ExecuteInner()
        {
            return true;
        }
    }
}
{% endhighlight %}

 As for the actual implementation, we won't go into too much detail but **focus on the most important bits**. MSBuild calls the `Execute` method which will later delegate to `ExecuteInner` and its return value signals whether the task succeeded or failed. The inherited `BuildEngine` property allows us to log information, warning, and error messages. Users of the task can opt-in to treat warnings of the task as errors by setting the `TreatWarningsAsErrors` property. The `StringParameter` property is a required input value. The `FilesParameter` item group is optional and can contain a list of files. In many situations, a `ITaskItem` can also be a ordinary string value, like for `PackageReference`.

## Wiring the Task

In this next step we'll **wire up the task implementation in a `.targets` file**, which will be included in our NuGet package and automatically loaded from a referencing project. In this file – here `CustomTasks.targets` we'll load the task assembly, create a new XML task element, define a couple default values, and create a new target that calls the task:

{% highlight xml linenos %}
<?xml version="1.0" encoding="utf-8"?>
<Project ToolsVersion="4.0" DefaultTargets="Build" xmlns="http://schemas.microsoft.com/developer/msbuild/2003">

  <PropertyGroup>
    <CustomTasksAssembly>$(MSBuildThisFileDirectory)\$(MSBuildThisFileName).dll</CustomTasksAssembly>
  </PropertyGroup>

  <UsingTask TaskName="$(MSBuildThisFileName).CustomTask" AssemblyFile="$(CustomTasksAssembly)"/>

  <!-- Default Properties -->
  <PropertyGroup>
    <CustomTaskContinueOnError Condition="'$(CustomTaskContinueOnError)' == ''">False</CustomTaskContinueOnError>
    <CustomTaskStringParameter Condition="'$(CustomTaskStringParameter)' == ''">Default Value</CustomTaskStringParameter>
  </PropertyGroup>

  <Target Name="RunCustomTask" BeforeTargets="CoreCompile">
    <CustomTask
            ContinueOnError="$(CustomTaskContinueOnError)"
            StringParameter="$(CustomTaskStringParameter)"
            FilesParameter="@(CustomTaskFilesParameter)"/>
  </Target>

</Project>
{% endhighlight %}

Defining the `CustomTasksAssembly` (Line 5) does not only help us to not repeat ourselves when we reference multiple tasks from the same assembly (Line 8), but is also great for debugging, as we'll see later. Also note that we're using a couple of [well-known MSBuild properties](https://docs.microsoft.com/visualstudio/msbuild/msbuild-reserved-and-well-known-properties) like `MSBuildThisFileDirectory` and `MSBuildThisFileName` to avoid magic strings being scattered around our file. Following best practices makes renaming or relocating the task more effortless! The task invocation also uses the `ContinueOnError` property (Line 18) – one of the [common properties available to all task elements](https://docs.microsoft.com/visualstudio/msbuild/task-element-msbuild). 

## Creating the NuGet Package

As already mentioned, we won't pack the MSBuild tasks project directly, but have another project `MainLibrary` **include the task infrastructure in its package**. In order to load the `CustomTasks.targets` file from the previous section, we create another `MainLibrary.props` and `MainLibrary.targets` file in our `MainLibrary` project:

{% highlight xml linenos %}
<?xml version="1.0" encoding="utf-8"?>
<Project ToolsVersion="4.0" DefaultTargets="Build" xmlns="http://schemas.microsoft.com/developer/msbuild/2003">

  <PropertyGroup>
    <CustomTasksDirectory Condition="'$(MSBuildRuntimeType)' == 'Core'">$(MSBuildThisFileDirectory)\netcore</CustomTasksDirectory>
    <CustomTasksDirectory Condition="'$(MSBuildRuntimeType)' != 'Core'">$(MSBuildThisFileDirectory)\netfx</CustomTasksDirectory>
  </PropertyGroup>

</Project>
{% endhighlight %}

In the `.props` file, we're defining a property `CustomTasksDirectory` that points to the directory containing our task. Note that we need to **take the MSBuild runtime into account** by checking `MSBuildRuntimeType` (Line 5-6). A project targeting `netcoreapp2.1` would still use .NET Framework for running MSBuild inside Visual Studio, while the same project would use MSBuild for .NET Core when compiling via `dotnet build` from the command-line. I.e., it must not be subject to [framework version folder structure](https://docs.microsoft.com/en-us/nuget/create-packages/supporting-multiple-target-frameworks#framework-version-folder-structure). In the `.targets` file, we will import the `CustomTasks.targets` file: 

{% highlight xml linenos %}
<?xml version="1.0" encoding="utf-8"?>
<Project ToolsVersion="4.0" DefaultTargets="Build" xmlns="http://schemas.microsoft.com/developer/msbuild/2003">

  <Import Project="$(CustomTasksDirectory)\CustomTasks.targets" Condition="'$(CustomTasksEnabled)' != 'False'" />

</Project>
{% endhighlight %}

We also add a condition to check if `CustomTasksEnabled` is enabled (Line 4). This little trick allows us to **easily opt-out from attaching the task**. Faulty MSBuild tasks have a high chance to completely break a referencing project and to cause confusion about where the error originates from:

<div class="tweet" tweetID="1072394313592647684">I really try to like @JetBrainsRider - but I just don’t have enough time for it...</div>

As one of the last steps, we **define the package structure** in our `MainLibrary.csproj` project file:

{% highlight xml linenos %}
<ItemGroup Condition="'$(TargetFramework)' == ''">
  <None Include="$(MSBuildProjectName).props" PackagePath="build" Pack="true"/>
  <None Include="$(MSBuildProjectName).targets" PackagePath="build" Pack="true"/>
  <None Include="..\CustomTasks\CustomTasks.targets" PackagePath="build\netcore" Pack="true"/>
  <None Include="..\CustomTasks\CustomTasks.targets" PackagePath="build\netfx" Pack="true"/>
  <None Include="..\CustomTasks\bin\$(Configuration)\netcoreapp2.1\publish\**\*.*" PackagePath="build\netcore" Pack="true"/>
  <None Include="..\CustomTasks\bin\$(Configuration)\net472\publish\**\*.*" PackagePath="build\netfx" Pack="true"/>
</ItemGroup>
{% endhighlight %}

It is important to remember, that to properly create the package, we first need to call `dotnet publish` for the supported target frameworks. The complete **list of invocations** should be as follows: 

{% highlight batch linenos %}
dotnet publish --framework netcoreapp2.1
dotnet publish --framework net472
dotnet pack
{% endhighlight %}

## Debugging a Task

Let's get to the most interesting part of how we can **effectively debug MSBuild integration** in a test project:

<div class="tweet" tweetID="1183638078767927297">Is there a decent workflow for writing MSBuild tasks? Finally looking at adding a proper MSBuild task to FunctionMonkey (it currently uses a console app - that'll stay too). Writing it seems straightforward.... debugging it looks like it might be painful.</div>

Indeed, with the **console application approach**, things are rather easy. On Windows we can call `System.Diagnostics.Debugger.Launch()`, which **fires up the Just-In-Time Debugger** so that we can attach to the process. By default, this will use the [Visual Studio JIT Debugger](https://docs.microsoft.com/en-us/visualstudio/debugger/debug-using-the-just-in-time-debugger?view=vs-2019), but we can also [configure JetBrains Rider as the Just-In-Time Debugger](https://blog.jetbrains.com/dotnet/2019/04/16/edit-continue-just-time-debugging-debugger-improvements-rider-2019-1/). As of now, this strategy is [not supported on Linux/macOS](https://github.com/dotnet/runtime/issues/38427). The best workaround I've found is to call `SpinWait.SpinUntil(() => Debugger.IsAttached)`, which will wait until the debugger is actually attached.

Taking the **custom MSBuild task approach**, we have a bit more footwork to do. Referencing a package via `PackageReference`, their main `.props` and `.targets` files get [automatically included](https://docs.microsoft.com/nuget/create-packages/creating-a-package#include-msbuild-props-and-targets-in-a-package). This is not the case when using `ProjectReference` on the same project! We could create an actual package, but that has the unpleasant side-effect of getting persisted in our global NuGet cache:

<div class="tweet" tweetID="965325828455321600">I want to point to to a nupckg file directly ideally. The problem with local feeds is that packages get cached.</div>

Deleting from the cache, or incrementing versions – all those are rather poor workarounds compared to a possible [better development and testing experience](https://github.com/NuGet/Home/issues/6579) for NuGet packages. A better alternative is to **manually wire up our test project**:

{% highlight xml linenos %}
<Project Sdk="Microsoft.NET.Sdk">

  <Import Project="..\MainLibrary\MainLibrary.props"/>

  <PropertyGroup>
    <TargetFramework>netcoreapp2.1</TargetFramework>
    <OutputType>Exe</OutputType>
    <TargetFramework>netcoreapp3.1</TargetFramework>
    <CustomTasksEnabled Condition="'$(CustomTasksEnabled)' == ''">False</CustomTasksEnabled>
    <CustomTasksDirectory>$(MSBuildThisFileDirectory)\..\CustomTasks\bin\Debug\netcoreapp2.1\publish</CustomTasksDirectory>
  </PropertyGroup>

  <ItemGroup>
    <ProjectReference Include="..\MainLibrary\MainLibrary.csproj"/>
  </ItemGroup>

  <Import Project="..\MainLibrary\MainLibrary.targets"/>

</Project>
{% endhighlight %}

The **`.props` file should be imported at the very beginning** of the project file (Line 3), allowing to set default property values. Meanwhile, the **`.targets` file should be important at the very end** of the project file (Line 16), to ensure the task is run with the relevant input values. Here, we will also utilize our trick from the previous section, and set a default for `CustomTasksEnabled` (Line 8), which can either be overridden from the outside (hence the condition) or manually changed to **test behavior in the IDE**. We should also not forget about adjusting the `CustomTasksDirectory` property to **point to the local version** of our task.

With [JetBrains Rider](https://jetbrains.com/rider) we can use [run configurations](https://www.jetbrains.com/help/rider/Run_Debug_Configuration.html) to make this process more convenient. In the first configuration `Publish CustomTasks`, we will **publish the MSBuild task project**:

![Publishing MSBuild Tasks via Run Configuration](/assets/images/2020-08-04-implementing-and-debugging-custom-msbuild-tasks/run-configuration-publish.png){:width="750px" .shadow}

In a second configuration `Run CustomTasks` we will depend on the first one (see _Before launch_), and call `MSBuild.dll /t:Clean;Restore;Pack /p:CustomTasksEnabled=True` to **invoke MSBuild on the test project**:

![Running MSBuild Tasks via Run Configuration](/assets/images/2020-08-04-implementing-and-debugging-custom-msbuild-tasks/run-configuration-run.png){:width="750px" .shadow}

Note that this is the place when we need to override the `CustomTasksEnabled` property to execute our task. Also we should **make an educated choice of which targets should be invoked**. If our task integrates with the `Restore` target, then we really need to execute `Clean` before, because otherwise it may be skipped for consecutive builds.

Now that we're all set up, we can use the `Run CustomTasks` configuration to finally debug our task implementation:

![Running Custom MSBuild Tasks via Run Configuration](/assets/images/2020-08-04-implementing-and-debugging-custom-msbuild-tasks/debug.gif){:width="750px" .shadow}

## Troubleshooting MSBuild

Sooner or later we will run into issues with our MSBuild integration, especially regarding the `.props` and `.targets` files. We might reference a wrong property, forget about escaping, or just have a typo in our identifiers. The _Project Properties_ dialog is a good place to start investigations and to see **evaluated properties and imports** for a project file:

<div class="swiper swiper-demo swiper-demo--image" style="max-width: 650px">
  <div class="swiper__wrapper">
    <div class="swiper__slide"><img class="lightbox-ignore" src="../../../../assets/images/2020-08-04-implementing-and-debugging-custom-msbuild-tasks/project-properties.png"/></div>
    <div class="swiper__slide"><img class="lightbox-ignore" src="../../../../assets/images/2020-08-04-implementing-and-debugging-custom-msbuild-tasks/project-imports.png"/></div>
  </div>
  <div class="swiper__button swiper__button--prev fas fa-chevron-left"></div>
  <div class="swiper__button swiper__button--next fas fa-chevron-right"></div>
</div>

Using the common keyboard shortcut, we can also easily **copy values from grid cells**. If we need even more insight, then the [MSBuild Structured Log Viewer](https://msbuildlog.com/) created by [Kirill Osenkov](https://twitter.com/KirillOsenkov) can be of great help:

<div class="tweet" tweetID="1284026601416654848">2 years ago I rewrote our entire build pipeline in mostly msbuild. Once I learned about structured log viewer my estimations were cut in half. MSBuild has become a lot more of a regular programming task since then.</div>

The Structured Log Viewer operates on [binary log files](https://docs.microsoft.com/visualstudio/msbuild/obtaining-build-logs-with-msbuild#save-a-binary-log), which can be created by passing `/binaryLogger:output.binlog` to the MSBuild invocation. Binary log files provide the **highest level of completeness and verbosity**, even compared to most-diagnostic level for text logs. Imports of files and execution of targets are **hierarchically visualized using a tree view**. Particularly when trying to find a proper [target build order](https://docs.microsoft.com/visualstudio/msbuild/target-build-order) for our integration, we can easily check on the **flattened temporal order view**:

<div class="tweet" tweetID="1192209843630665728">The latest MSBuild Log Viewer adds a new option to display targets in one flat list chronologically, it may be easier to see in which order the targets actually ran:</div>

Another benefit is that when testing our task on large projects that imply a time-consuming compilation, we can **replay a single task** without executing all its dependencies:

<div class="tweet" tweetID="1030638751951638529">Soon in MSBuild Structured Log Viewer: run or debug MSBuild tasks by using the exact parameter values from the binlog! "Replay" tasks in isolation outside of the build.</div>

For developers on Windows there is a **special surprise in JetBrains Rider**! We can right-click the test project, choose _Advanced Build Actions_ and execute _Rebuild Selected Projects with Diagnostics_:

![Running Custom MSBuild Tasks via Run Configuration](/assets/images/2020-08-04-implementing-and-debugging-custom-msbuild-tasks/structured-log-viewer.gif){:.shadow}

Given that Structured Log Viewer can already run on [Avalonia UI](https://github.com/AvaloniaUI/Avalonia/), maybe we can see the same feature also for macOS and Linux soon.

## Acknowledgements

I want to add that much of my adventures with MSBuild are only of good nature and happy endings because my friend [Martin Ullrich](https://twitter.com/dasmulli) is such an **MSBuild wizard**. If you don't follow him yet, you really should. Sorry Martin for sending more MSBuild enthusiasts your way! 🤗

