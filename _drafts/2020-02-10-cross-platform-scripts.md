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

https://stackoverflow.com/questions/17510688/single-script-to-run-in-both-windows-batch-and-linux-bash
[CMDs label syntax as comment marker](https://www.robvanderwoude.com/comments.php)

Why is a single entry script a good idea?
- Only one file
- Simplified documentation
- Simplified "matrix" approach
- Even with PowerShell Core, not everything is cross-platform
- I.e., dotnet-install.ps1
- From my own experience, invoking scripts from CLI is more robust towards encoding


For small scripts, this is very attractive. For more complex scripts, we can delegate work to a proper Bash or PowerShell script.

{% highlight batch linenos %}
:; set -eo pipefail
:; ./build.sh "$@"
:; exit $?

@ECHO OFF
powershell -ExecutionPolicy ByPass -NoProfile %0\..\build.ps1 %*
{% endhighlight %}

- `chmod +x build.cmd`
- `git update-index --add --chmod=+x build.cmd`
- `svn propset svn:executable on build.cmd`

Georgs blog post

https://plugins.jetbrains.com/plugin/10249-powershell
https://plugins.jetbrains.com/plugin/index?xmlId=BashSupport
https://plugins.jetbrains.com/plugin/4230-bashsupport/

Show renaming, inspections, find usages, debugging(?), commenting

[setup.sh](https://github.com/nuke-build/nuke/blob/1b5070b72cea385dad12c43c392e0b7992936bc9/bootstrapping/setup.sh)
[setup.ps1](https://github.com/nuke-build/nuke/blob/1b5070b72cea385dad12c43c392e0b7992936bc9/bootstrapping/setup.ps1)

Writing those scripts and keeping them in sync was extremely hard and nerve-racking.
