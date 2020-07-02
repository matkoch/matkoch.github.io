---
title: The Road to Cross-Platform Setup & Bootstrapping in .NET
tags:
- .NET
- Bash
- JetBrains Rider
- PowerShell
- Windows Batch
mode: immersive
header:
  theme: dark
article_header:
  type: overlay
  theme: dark
  background_color: '#203028'
  background_image:
    gradient: 'linear-gradient(135deg, rgba(34, 0, 170 , .2), rgba(139, 34, 139, .2))'
    src: assets/images/2020-07-01-road-to-cross-platform-setup-and-bootstrapping-in-dotnet/cover.jpg
twitter_card: assets/images/2020-07-01-road-to-cross-platform-setup-and-bootstrapping-in-dotnet/thumbnail.jpeg
---

***TL;DR: PowerShell and Bash scripts are indispensable for developers. While benefiting from being transparent compared to executables, they're not exactly easy to read by users nor easy to write by maintainers, and also suffer from duplication for different operating systems. With the .NET Core SDK, this issue is mostly solved using .NET Global Tools. Nevertheless, global tools come at the cost of requiring the SDK to be pre-installed. Several workarounds for local and build server environments exist, however, shell scripts are really the only solid solution. Single-entry scripts can help to erase the last relics from PowerShell and Bash scripts for documentation and end-users.***

When I started with [NUKE](https://nuke.build), part of the hardest work was to provide a convenient setup experience. NUKE was one of the first build systems to use conventional [console applications](https://docs.microsoft.com/en-us/dotnet/core/tutorials/with-visual-studio-code) in order to support first-class development experience. [BullsEye](https://github.com/adamralph/bullseye) and [FlubuCore](https://github.com/dotnetcore/FlubuCore) followed shortly after. Noteworthy, they come in a more simple form where the build project is isolated from the main solution by default. In contrast but not necessarily, NUKE allows us to **add the build project to the solution** besides other projects to make it more accessible. This approach works really great, but it does require some **tedious tweaks that are good to automate**. Another natural consequence of using build projects, is that they **require a .NET Core SDK installation** to build and run, which is crucial for build servers and new project contributors.

In the next sections, we'll look at how the setup experience in NUKE evolved from complex PowerShell and Bash scripts to using .NET Global Tools, and how the .NET Core bootstrapping via shell scripts makes everything more cross-platform and self-contained.

## Hacking Shell Scripts

Back in 2017, in the times of `netstandard1.6`, .NET Core was still on its journey of getting widely adopted in the .NET ecosystem. It was also the time when I made my first steps with [JetBrains Rider](https://jetbrains.com/rider), and moved to macOS for the major part of my development. But of course, many developers still relied on the .NET Framework, so for me it was crucial to make NUKE's setup process to support both platforms. Nobody likes to **download and run arbitrary executables from the web**, especially when the projet is barely known. Therefore, I went with the more **transparent PowerShell and Bash scripts**, which can be easily investigated before execution:

{% highlight batch linenos %}
powershell -Command iwr https://nuke.build/powershell -OutFile setup.ps1
powershell -ExecutionPolicy ByPass -File .\setup.ps1

curl -Lsfo setup.sh https://nuke.build/bash
chmod +x setup.s
./setup.sh
{% endhighlight %}

Writing the PowerShell [setup.ps1](https://github.com/nuke-build/nuke/blob/1b5070b72cea385dad12c43c392e0b7992936bc9/bootstrapping/setup.ps1) script was mostly effortless, but the Bash [setup.sh](https://github.com/nuke-build/nuke/blob/1b5070b72cea385dad12c43c392e0b7992936bc9/bootstrapping/setup.sh) script had some **really horrible code** in it. For instance, downloading a file, replacing placeholders with actual values and writing everything to disk:

{% highlight batch linenos %}
sed -e 's~_TARGET_FRAMEWORK_~'"$TARGET_FRAMEWORK"'~g' \
    -e 's~_BUILD_PROJECT_GUID_~'"$PROJECT_GUID"'~g' \
    -e 's~_BUILD_PROJECT_NAME_~'"$BUILD_PROJECT_NAME"'~g' \
    -e 's~_SOLUTION_DIRECTORY_~'"${SOLUTION_DIRECTORY_RELATIVE//\//\\}"'~g' \
    -e 's~_NUKE_VERSION_~'"$NUKE_VERSION"'~g' \
    -e 's~_NUKE_VERSION_MAJOR_MINOR_~'"${NUKE_VERSION_PARTS[0]}.${NUKE_VERSION_PARTS[1]}"'~g' \
    <<<"$(curl -Lsf $BOOTSTRAPPING_URL/.build.$PROJECT_FORMAT.csproj)" \
    > "$BUILD_PROJECT_FILE"
{% endhighlight %}

Another example, that scans an existing file for specific lines, and adds additional lines after them:

{% highlight batch linenos %}
awk "/MinimumVisualStudioVersion/{print \$0 RS \"$PROJECT_DEFINITION\";next}1" "$SOLUTION_FILE" > "$SOLUTION_FILE.bak"
awk "/ProjectConfigurationPlatforms/{print \$0 RS \"$PROJECT_CONFIGURATION\";next}1" "$SOLUTION_FILE.bak" > "$SOLUTION_FILE"
{% endhighlight %}

Using the [Bash support](https://plugins.jetbrains.com/plugin/4230-bashsupport) and [PowerShell support](https://plugins.jetbrains.com/plugin/10249-powershell) plugins for JetBrains Rider, made it a lot more bearable to work with those languages that I'm generally not fluent with. Especially, when it comes to syntax highlighting, code completion, navigation, renaming variables, fixing minor code issues, or quickly adding comments:

![PowerShell Support in JetBrains Rider](/assets/images/2020-07-01-road-to-cross-platform-setup-and-bootstrapping-in-dotnet/language-support.gif){:width="700px" .shadow}

Even though the plugins helped a lot, honestly, I can't say that I really enjoyed maintaining the setup scripts. I've **never felt familiar with the code** after coming back, and there were plenty of stupid and surprising issues, like [missing parentheses](https://github.com/nuke-build/nuke/pull/28) or [crashes if IE wasn't started once before](https://github.com/nuke-build/nuke/issues/11).

## Setup with Global Tools

With `netcoreapp2.1` the .NET Core SDK introduced [.NET Global Tools](https://docs.microsoft.com/en-us/dotnet/core/tools/global-tools) and since NUKE meanwhile established a reasonable community, I've decided that it's okay [moving towards a unified executable](https://github.com/nuke-build/nuke/issues/26):

![Moving to Global Tools](/assets/images/2020-07-01-road-to-cross-platform-setup-and-bootstrapping-in-dotnet/global-tools.png){:width="600px" .shadow}

Global tools are built from a single source and can be installed and executed on Windows, Linux and macOS. So no matter what the environment was, the setup experience for NUKE became as easy as:

{% highlight batch linenos %}
dotnet tool install Nuke.GlobalTool --global
nuke :setup
{% endhighlight %}

That day the [shell scripts have been replaced by a global tool](https://github.com/nuke-build/nuke/commit/2481d251d9d4a7b090632a879a0c93221c146ee0), I could delete 400 LOC that were basically duplicated and hard to maintain, and replace them with roughly 300 lines of clean C# code.

## Bootstrapping .NET Core SDK

Another difference compared to BullsEye and FlubuCore, is that NUKE provides bootstrapping scripts that will install the .NET Core SDK without any further requirements. As an example, here is the `build.sh` script:

{% highlight bash linenos %}
#!/usr/bin/env bash

echo $(bash --version 2>&1 | head -n 1)

set -eo pipefail
SCRIPT_DIR=$(cd "$( dirname "${BASH_SOURCE[0]}" )" && pwd)

###########################################################################
# CONFIGURATION
###########################################################################

BUILD_PROJECT_FILE="$SCRIPT_DIR/_BUILD_DIRECTORY_/_BUILD_PROJECT_NAME_.csproj"
TEMP_DIRECTORY="$SCRIPT_DIR/_ROOT_DIRECTORY_/.tmp"

DOTNET_GLOBAL_FILE="$SCRIPT_DIR/_ROOT_DIRECTORY_/global.json"
DOTNET_INSTALL_URL="https://dot.net/v1/dotnet-install.sh"
DOTNET_CHANNEL="Current"

export DOTNET_CLI_TELEMETRY_OPTOUT=1
export DOTNET_SKIP_FIRST_TIME_EXPERIENCE=1

###########################################################################
# EXECUTION
###########################################################################

function FirstJsonValue {
    perl -nle 'print $1 if m{"'$1'": "([^"]+)",?}' <<< ${@:2}
}

# If global.json exists, load expected version
if [[ -f "$DOTNET_GLOBAL_FILE" ]]; then
    DOTNET_VERSION=$(FirstJsonValue "version" $(cat "$DOTNET_GLOBAL_FILE"))
    if [[ "$DOTNET_VERSION" == ""  ]]; then
        unset DOTNET_VERSION
    fi
fi

# If dotnet is installed locally, and expected version is not set or installation matches the expected version
if [[ -x "$(command -v dotnet)" && (-z ${DOTNET_VERSION+x} || $(dotnet --version 2>&1) == "$DOTNET_VERSION") ]]; then
    export DOTNET_EXE="$(command -v dotnet)"
else
    DOTNET_DIRECTORY="$TEMP_DIRECTORY/dotnet-unix"
    export DOTNET_EXE="$DOTNET_DIRECTORY/dotnet"

    # Download install script
    DOTNET_INSTALL_FILE="$TEMP_DIRECTORY/dotnet-install.sh"
    mkdir -p "$TEMP_DIRECTORY"
    curl -Lsfo "$DOTNET_INSTALL_FILE" "$DOTNET_INSTALL_URL"
    chmod +x "$DOTNET_INSTALL_FILE"

    # Install by channel or version
    if [[ -z ${DOTNET_VERSION+x} ]]; then
        "$DOTNET_INSTALL_FILE" --install-dir "$DOTNET_DIRECTORY" --channel "$DOTNET_CHANNEL" --no-path
    else
        "$DOTNET_INSTALL_FILE" --install-dir "$DOTNET_DIRECTORY" --version "$DOTNET_VERSION" --no-path
    fi
fi

echo "Microsoft (R) .NET Core SDK version $("$DOTNET_EXE" --version)"

"$DOTNET_EXE" build "$BUILD_PROJECT_FILE" /nodeReuse:false -nologo -clp:NoSummary --verbosity quiet
"$DOTNET_EXE" run --project "$BUILD_PROJECT_FILE" --no-build -- "$@"
{% endhighlight %}

The bootstrapping scripts are very important to run on build servers, since the required .NET Core SDK might not be installed. Now, many will say that there is the [_Use .NET Core_ task](https://docs.microsoft.com/en-us/azure/devops/pipelines/tasks/tool/dotnet-core-tool-installer?view=azure-devops) that will perform an [install according to the `global.json`](https://docs.microsoft.com/en-us/dotnet/core/tools/global-json?tabs=netcore3x#globaljson-schema) file, but this approach has several disadvantages. Firstly, a similar task **might not exist on the particular build server** we're using. Also, predefined tasks tend to be **very inflexible** and eventually force us to use [hacky workarounds](https://github.com/actions/setup-dotnet/issues/25#issuecomment-646925506) or reconsider going back to shell scripts:

<div class="tweet" tweetID="1274641221693169664">I cannot believe that it is not officially supported by Github actions to have multiple versions of the .NET Core SDK installed.</div>

Even more importantly, with the rapid evolution of C# as a language, chances are good that we might not have the required SDK installed for a repository we've just forked. [Joseph Woodward](https://twitter.com/joe_mighty) came up with the [InstallSdkGlobalTool](https://www.nuget.org/packages/installsdkglobaltool) to partially solve that for local machines at least. So in a repository with `global.json` file, we can just invoke `dotnet-install-sdk` and it will do its magic:

<div class="tweet" tweetID="1169028787427827712">Blogged: Managing your .NET Core SDK versions with the .NET Install SDK Global Tool</div>

Remember that this requires at least some .NET Core SDK to be installed. Also keep in mind that this is a manual step and the tool needs to be installed on our colleagues or contributors machine.

<!--
- https://github.com/cake-build/cake/blob/0eaee7e2a7d95611b1de58b40493f550ced31c78/build.ps1
- https://github.com/cake-build/cake/blob/0eaee7e2a7d95611b1de58b40493f550ced31c78/build.sh
-->

## Single-Entry Scripts

So far, the experience was looking quite good already. But one thing that always bothered me, was that `build.sh` and `build.ps1` were exclusive to UNIX and Windows systems. Can't we have a single-entry script that can be invoked cross-platform? That would solve the following issues:

- **Documentation** – [Duplicated invocation snippets](https://github.com/dotnet/maui/blob/85ee7fe836dc005d44eb0ba6b4660b1c6341d59c/build.cake#L9-L15) for both `build.sh` and `build.ps1` are a waste of time while reading, annoying when writing documentation, and also more prone to becoming outdated.
- **Multi-platform pipelines** – Many build servers enable us to write [multi-platform pipelines](https://docs.microsoft.com/en-us/azure/devops/pipelines/get-started-multiplatform) where a set of common build steps is executed for different operating systems using a matrix approach. This works great with pre-defined tasks, but it falls short when we need to [call different scripts](https://docs.microsoft.com/en-us/azure/devops/pipelines/get-started-multiplatform).

Based on a [Stack Overflow question](https://stackoverflow.com/questions/17510688/single-script-to-run-in-both-windows-batch-and-linux-bash), I've found that the `:` colon character can be used to effectively skip code in Batch scripts, while POSIX shells simply evaluate them to `true`:
 
{% highlight batch linenos %}
:; set -eo pipefail
:; SCRIPT_DIR=$(cd "$( dirname "${BASH_SOURCE[0]}" )" && pwd)
:; ${SCRIPT_DIR}/build.sh "$@"
:; exit $?

@ECHO OFF
powershell -ExecutionPolicy ByPass -NoProfile %0\..\build.ps1 %*
{% endhighlight %}

Depending on our current operating system, calling `build.cmd` will either delegate to `build.sh` or `build.ps1`. It's probably obvious, that for larger and more complex scripts, this approach doesn't scale very well. **Most of the IDE tooling won't work** apart from very simple [hippie completion](https://www.jetbrains.com/help/idea/auto-completing-code.html#hippie_completion).

After some time [Georg Dangl](https://twitter.com/georgdangl) discovered an issue that the [generated scripts are not executable by default](https://blog.dangl.me/archive/executing-nuke-build-scripts-on-linux-machines-with-correct-file-permissions/). When invoking the scripts, the error message `permission denied: ./build.cmd` was returned. Meanwhile, this has been addressed so that the setup in NUKE now executes the following commands:

- `chmod +x build.cmd` – Allow local execution
- `git update-index --add --chmod=+x build.cmd` – Mark file to be executable in Git
- `svn propset svn:executable on build.cmd` – Mark file to be executable in Subversion

## Conclusion

With .NET Core running cross-platform, it is important to ensure shell scripts to work cross-platform as well. Setup routines are perfectly qualified to be moved into global tools, whereas bootstrapping tasks, like bootstrapping the .NET Core SDK, should remain in PowerShell and Bash scripts to improve the experience for build servers and first-time contributors.

 **Bootstrap your builds frictionless with [NUKE](https://nuke.build)!**
