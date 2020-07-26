## When it's totally adequate
- Hook into compilation process (link)
    - Adding files to compilation
    - Rewriting IL Code
    - Generating from source
- Directory.Build.props/targets
    - Project/Package References (link)
    - Metadata

## When MSBuild can be hard

- No intellisense, no debugging
- Even seasoned developers can quickly run into 

<div class="tweet" tweetID="1283822281483939840">Anyone have any idea what happened to the semicolon and the spaces in this msbuild task?
</div>

- If possible, it's best to leave the project system alone. There's a high likelihood that something doesn't work on another system. -> This is not to say that there are no use cases for doing so.

- Failures often cause uncertainty where the issue really is

<div class="tweet" tweetID="1072394313592647684">I really try to like @JetBrainsRider - but I just don’t have enough time for it...</div>

- It's an anti-pattern!

## .NET CLI and Shell Scripts

- Both of the above cases can easily be solved without messing with the project system

- For instance, we can easily determine version numbers by using GitVersion just as a CLI tool, and pass captured values from the CLI `dotnet build -p:AssemblyVersion=` to [generate the assembly info](https://docs.microsoft.com/en-us/dotnet/core/tools/csproj#assemblyinfo-properties)

- [Support for `@(InternalsVisibleTo)` items](https://github.com/dotnet/sdk/pull/3439)
- [Support for `@(AssemblyMetadata)` items](https://github.com/dotnet/sdk/pull/3440)


https://twitter.com/randompunter/status/1274768021312155648
@randompunter: Control your build environment, don't rely on others', and make it portable so it's exactly the same locally as in CI - build in a container. Doing tons of stuff in a workflow yaml is a smell imho and not fun running, testing, debugging it locally.

https://twitter.com/bradwilson/status/1219653745090293760
@bradwilson: For me, ./build is not about "does dotnet build work?" but about "how do you do everything else, after dotnet build". Like... packaging, signing, uploading.
