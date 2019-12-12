---
title: Replacing Package References with Project References
tags:
- MSBuild
- .NET
- JetBrains Rider
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
    src: assets/images/2019-12-12-replace-package-with-project-references.jpg
---

Working on multi-repository projects can be difficult if there are NuGet package dependencies involved that need to


<div id="tweet" tweetID="515490786800963584"></div>

<!--more-->
{% highlight xml linenos %}
<?xml version="1.0" encoding="utf-8"?>
<Project ToolsVersion="4.0" DefaultTargets="Build" xmlns="http://schemas.microsoft.com/developer/msbuild/2003">

    <PropertyGroup>
        <ReplacePackageReferences Condition="$(ReplacePackageReferences) == ''">True</ReplacePackageReferences>
        <SolutionPath>$(MSBuildThisFileDirectory)\global.sln</SolutionPath>
    </PropertyGroup>

    <Choose>
        <When Condition="$(ReplacePackageReferences) AND '$(SolutionPath)' != '' AND '$(SolutionPath)' != '*undefined*' AND Exists('$(SolutionPath)')">

        <PropertyGroup>
            <SolutionFileContent>$([System.IO.File]::ReadAllText($(SolutionPath)))</SolutionFileContent>
            <SmartSolutionDir>$([System.IO.Path]::GetDirectoryName( $(SolutionPath) ))</SmartSolutionDir>
            <RegexPattern>(?&lt;="[PackageName]", ")(.*)(?=", ")</RegexPattern>
        </PropertyGroup>

        <ItemGroup>
            <!-- Keep the identity of the package reference -->
            <SmartPackageReference Include="@(PackageReference)">
            <PackageName>%(Identity)</PackageName>
            <InSolution>$(SolutionFileContent.Contains('\%(Identity).csproj'))</InSolution>
            </SmartPackageReference>

            <!-- Filter them by mapping them to another itemGroup using the WithMetadataValue item function -->
            <PackageInSolution Include="@(SmartPackageReference -> WithMetadataValue('InSolution', True) )">
            <Pattern>$(RegexPattern.Replace('[PackageName]','%(PackageName)') )</Pattern>
            <SmartPath>$([System.Text.RegularExpressions.Regex]::Match( '$(SolutionFileContent)', '%(Pattern)' ))</SmartPath>
            </PackageInSolution>

            <ProjectReference  Include="@(PackageInSolution -> '$(SmartSolutionDir)\%(SmartPath)' )"/>

            <!-- Remove the package references that are now referenced as projects -->
            <PackageReference Remove="@(PackageInSolution -> '%(PackageName)' )"/>
        </ItemGroup>

        </When>
    </Choose>

</Project>
{% endhighlight %}
