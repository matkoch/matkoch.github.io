{% highlight xml linenos %}
<?xml version="1.0" encoding="utf-8"?>
<Project ToolsVersion="4.0" DefaultTargets="Build" xmlns="http://schemas.microsoft.com/developer/msbuild/2003">

    <PropertyGroup>
        <!-- TODO: adapt global solution file -->
        <SolutionPath>$(MSBuildThisFileDirectory)\Global.sln</SolutionPath>
        <ReplacePackageReferences Condition="$(ReplacePackageReferences) == ''">True</ReplacePackageReferences>
    </PropertyGroup>

    <Choose>
        <When Condition="$(ReplacePackageReferences) AND '$(SolutionPath)' != '' AND '$(SolutionPath)' != '*undefined*' AND Exists('$(SolutionPath)')">

        <PropertyGroup>
            <SolutionFileContent>$([System.IO.File]::ReadAllText($(SolutionPath)))</SolutionFileContent>
            <SmartSolutionDir>$([System.IO.Path]::GetDirectoryName( $(SolutionPath) ))</SmartSolutionDir>
            <RegexPattern>(?&lt;="[PackageName]", ")(.*)(?=", ")</RegexPattern>
        </PropertyGroup>

        <ItemGroup>
            <SmartPackageReference Include="@(PackageReference)">
                <PackageName>%(Identity)</PackageName>
                <InSolution>$(SolutionFileContent.Contains('\%(Identity).csproj'))</InSolution>
            </SmartPackageReference>
        </ItemGroup>

        <ItemGroup>
            <PackageInSolution Include="@(SmartPackageReference -> WithMetadataValue('InSolution', True) )">
                <Pattern>$(RegexPattern.Replace('[PackageName]','%(PackageName)') )</Pattern>
                <SmartPath>$([System.Text.RegularExpressions.Regex]::Match( '$(SolutionFileContent)', '%(Pattern)' ))</SmartPath>
            </PackageInSolution>
        </ItemGroup>

        <ItemGroup>
            <ProjectReference Include="@(PackageInSolution -> '$(SmartSolutionDir)\%(SmartPath)' )"/>
            <PackageReference Remove="@(PackageInSolution -> '%(PackageName)' )"/>
        </ItemGroup>

        </When>
    </Choose>

</Project>
{% endhighlight %}