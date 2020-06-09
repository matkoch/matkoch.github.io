---
title: Writing Cross-Platform Scripts
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
    src: assets/images/2020-02-10-cross-platform-scripts/cover.jpg
---

When I started with [NUKE](https://nuke.build), part of the hardest work was to provide a convenient setup experience. NUKE was one of the first build systems to use conventional [console applications](https://docs.microsoft.com/en-us/dotnet/core/tutorials/with-visual-studio-code) in order to support first-class development experience, including navigation, debugging, and refactorings. [BullsEye](https://github.com/adamralph/bullseye) and [FlubuCore](https://github.com/dotnetcore/FlubuCore) followed shortly after. However, they came in a more simple form where the build project is isolated from the main solution by default. In contrast (but not as requirement), NUKE allows to add the build project to our solution besides other projects to make it more accessible. This approach works really great, but it does require some tweaks that are good to automate. In the next sections, we'll look at how the setup experience evolved from complex PowerShell and Bash scripts to using .NET Global Tools, and how everything became more cross-platform and self-contained.

## Early .NET Core times

In the times of `netstandard1.6`, .NET Core was still on its rise of being widely adopted in the .NET ecosystem. Many developers still relied on the .NET Framework. 
 
 
 we still had to wait another year for global tools to arrive. Also, .NET Core on its own 


https://docs.microsoft.com/en-us/dotnet/core/tools/global-tools


[setup.sh](https://github.com/nuke-build/nuke/blob/1b5070b72cea385dad12c43c392e0b7992936bc9/bootstrapping/setup.sh)
[setup.ps1](https://github.com/nuke-build/nuke/blob/1b5070b72cea385dad12c43c392e0b7992936bc9/bootstrapping/setup.ps1)

https://stackoverflow.com/questions/17510688/single-script-to-run-in-both-windows-batch-and-linux-bash
[CMDs label syntax as comment marker](https://www.robvanderwoude.com/comments.php)

## Single-Entry Scripts

So far, the experience looks quite good already. However, one thing that always bothered me is that `build.sh` and `build.ps1` are exclusive to UNIX and Windows systems respectively.

Having a single-entry script can have great benefits for a lot of parties:

- **Documentation** – Duplicated invocation snippets for both `build.sh` and `build.ps1` are a waste of time while reading, annoying when writing documentation, and also more prone to becoming outdated.
- **Multi-platform pipelines** – Many build servers enable us to write [multi-platform pipelines](https://docs.microsoft.com/en-us/azure/devops/pipelines/get-started-multiplatform) where a set of common build steps is executed for different operating systems using a matrix approach. This works great with pre-defined tasks, but it falls short when we need to [call different scripts](https://docs.microsoft.com/en-us/azure/devops/pipelines/get-started-multiplatform).
- **Encoding** – I'm by far not an expert when it comes to encoding, but from my experience I had less issues when the native script interpreter was used for invocation.

I actually also considered [PowerShell Core](https://github.com/PowerShell/PowerShell) this time, but there is an unresolved issue that [`dotnet-install.ps1` is not supported on UNIX](https://github.com/dotnet/install-scripts/issues/23). Being able to execute this script is an essential requirement for building in environments that don't have the .NET Core SDK pre-installed, like build agents.
 
 I also recommend my other blog   


For small scripts, this is very attractive. For more complex scripts, we can delegate work to a proper Bash or PowerShell script.

{% highlight batch linenos %}
:; set -eo pipefail
:; ./build.sh "$@"
:; exit $?

@ECHO OFF
powershell -ExecutionPolicy ByPass -NoProfile %0\..\build.ps1 %*
{% endhighlight %}

## Executability

After some time [Georg Dangl](https://twitter.com/georgdangl) discovered an issue that the [generated scripts are not executable by default](https://blog.dangl.me/archive/executing-nuke-build-scripts-on-linux-machines-with-correct-file-permissions/). Meanwhile, this has been addressed so that the setup in NUKE now executes the following commands:

- `chmod +x build.cmd` – Allow local execution
- `git update-index --add --chmod=+x build.cmd` – Mark file to be executable in Git
- `svn propset svn:executable on build.cmd` – Mark file to be executable in Subversion

Georgs blog post

