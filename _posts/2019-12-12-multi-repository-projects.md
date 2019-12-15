---
title: 'Taming Multi-Repository Projects'
tags:
- .NET
- JetBrains Rider
- MSBuild
- NUKE
- VCS
hidden: true
mode: immersive
header:
  theme: dark
article_header:
  type: overlay
  theme: dark
  background_color: '#203028'
  background_image:
    gradient: 'linear-gradient(135deg, rgba(34, 139, 87 , .4), rgba(139, 34, 139, .4))'
    src: assets/images/2019-12-12-multi-repository-projects/cover.jpg
---

<style>
  .page__header .header__brand path {
    fill: rgba(255, 255, 255, .95);
  }
</style>


<!--more-->

Problem: You're working on a multi-repository project. Each of the repositories can release NuGet packages on its own. However, those repositories also define dependencies between each other – read core and leaf projects. In such cases, it's hard to keep an overview, not even to mention doing cross-repository refactorings. In this blog post we'll look at a few options how to effectively work with such architectures.

[Why not to use them?](https://github.com/dotnet/announcements/issues/119)
- Confusion for users
- PRs across repositories
- Inconsistencies/Duplicated efforts

Why use them?
- Isolated work
- Separate changelog
- GitVersion
- Different release cycles

## Generating Global Solutions

- Nice way to get all projects under one roof
- As I look at it, a global solution can be generated in two places
    - If there is a single more important repository -> generate it besides that
    - If all repositories are uniform -> create a new global repository
- For NUKE there is actually the nuke-build/nuke repository, which contains the code for the NuGet packages `Nuke.Common`, `Nuke.GlobalTool`, `Nuke.CodeGeneration`, and others
- So I'm generating a `nuke-global.sln` right beside the `nuke-common.sln`
- Due to this nature, I'm maintaining an `external/repositories.yml` file 

{%- include_relative 2019-12-12-multi-repository-projects/repositories-yml.md -%}

- Previously, more repositories for NSwag, Docker, Azure CLI, Kubernetes etc.
- Right now, actually only nuke-build/resharper and nuke-build/visualstudio depend on `Nuke.Common`
- Inside my NUKE build, I've added a target `CheckoutExternalRepositories` that takes care of cloning the listed repositories

{%- include_relative 2019-12-12-multi-repository-projects/checkout-target.md -%}

The target reads the `repositories.yml` file (line 9), converts all entries to `GitRepository` objects (line 10). While iterating the repositories, the final directory is determined (line 14). Based on the `UseSSH` parameter (line 1), we chose either `https://github.com/` or `git@github.com` as `origin` endpoint (line 15). If the directory doesn't exist yet, we're cloning it to the previously determined directory (line 18). Note that we're also directly checking out the _default branch_ for the repository. However, if the directory was already cloned, we're just updating the `origin` to the new one (line 21-22).

{%- include_relative 2019-12-12-multi-repository-projects/solution-target.md -%}


## Converting Package References

In most situations, having all projects in a single global solution is not going to solve the maintenance issue completely. Since we're dealing with multiple repositories, several projects in different repositories might have dependencies on each other in the form of NuGet package references. As you may know, there are two types for referencing NuGet packages:

- Using a [packages.config](https://docs.microsoft.com/en-us/nuget/reference/packages-config) is the old and verbose way. Project files usually become very bloated. Also, it transitive dependencies are flattened out, so it's hard to deal with updates.
- Using a [PackageReference](https://docs.microsoft.com/en-us/nuget/consume-packages/package-references-in-project-files) item group has become the standard since the [SDK-based project files](https://docs.microsoft.com/en-us/dotnet/core/tools/csproj) have been introduced. They mostly require just a single line to be added, make use of a common package cache by default, and are overall much more flexible to use.

I spent quite some time trying to find a way to conveniently convert package references into project references. Mostly, because there's also the requirement that the project file should _not_ be modified. That means, the original project file should stay intact, when I'm working on it via the local solution. Only when the project is used in our global solution, the reference types should be converted.

Take a deep breath in. There actually is a way using a `Directory.Build.targets` file to [customize our build](https://docs.microsoft.com/en-us/visualstudio/msbuild/customize-your-build). As a quick reminder, this file will be automatically included from the MSBuild engine when it's located in any parent directory to our project file. Your realize what that means? We can place this file _outside_ the other repositories into our repository containing the global solution. Send your thanksgivings to [Thomas Frenzel](https://github.com/dotnet/sdk/issues/1151#issuecomment-385133284) for this amazing approach:

{%- include_relative 2019-12-12-multi-repository-projects/directory-build.md -%}

To add a little bit of explanation:

|----------+-------------|
| Line(s)  | Explanation |
|:--------:|:------------|
| 6     | Defines the path to our global solution. The following logic is only applied when working in that particular solution. |
| 7     | Defines that by default all projects should have their package references replaced. We can individually opt-out by overriding in single project files. |
| 19-24 | Creates a new item group `SmartPackageReference` that knows whether the referenced package is available in the solution. |
| 26-31 | Creates another item group `PackageInSolution` based on the available packages and grabs their file paths. |
| 33-36 | Adds discovered projects as `ProjectReference` and removes them as `PackageReference`. |

**Known Limitations**

- `PackageId` must correspond to project name

## VCS Log and Operations

- VCS Root concept

![Statusbar Popup](/assets/images/2019-12-12-multi-repository-projects/statusbar.png){:height="400px" style="float: right"}