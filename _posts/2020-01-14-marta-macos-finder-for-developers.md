---
title: Marta - MacOS Finder for developers
key: marta-macos-finder-for-developers
tags:
- MacOS
- Tools
mode: immersive
header:
  theme: dark
article_header:
  type: overlay
  theme: dark
  background_color: '#203028'
  background_image:
    gradient: 'linear-gradient(135deg, rgba(34, 139, 87 , .1), rgba(139, 34, 139, .1))'
    src: assets/images/2020-01-14-marta/cover.jpg
---

Ever since I've fully switched from Windows to MacOS for my daily development, I was looking for a good Finder replacement. On Windows I was just using Windows Explorer, which wasn't great, but good enough. On MacOS however, the **native Finder application is far beyond being developer friendly**. It might be that I've missed to change some secret setting, but some of the **pain points** I had (back then) were:

- Copying paths from directories or files
- Navigating to paths from the clipboard
- No dual mode
- No tab support
- Directories and files are mixed in order

I tried several alternatives, like [TotalFinder](https://totalfinder.binaryage.com/), [muCommander](https://www.mucommander.com/), [PathFinder](https://cocoatech.com/), or [ForkLift](https://binarynights.com/). They all look quite polished, but very often I missed some ways of configuration, or it was too overloaded or too mouse-driven. Until today, I went with TotalFinder, as it was the tool that came closest to my requirements.

## Meet Marta

At some point, [Marta](https://marta.yanex.org/) popped up in my Twitter timeline. I don't remember who posted it. It might have been a colleague. Marta comes with dual mode (similar to [FarManager](https://www.farmanager.com/)), an integrated terminal, tab support, customizable columns, and DirStat (similar to [WinDirStat](https://windirstat.net/)):

![Marta overview](/assets/images/2020-01-14-marta/overview.png){:width="650px" .shadow}

## Surprising background

My first and foremost thought about Marta was: *Wow! This feels a bit like working in JetBrains Rider / IntelliJ IDEA.* That was mostly because of the way how you search and select file items. Just **type a search term**, which will be shown in the bottom right, and then **navigate with arrow keys** through the results:

![Searching and selecting](/assets/images/2020-01-14-marta/searching01.gif){:width="650px" .shadow}

Another reason is the **Lookup search**, which is shown in the upper middle of the application and allows to search for just anything:

![Lookup search](/assets/images/2020-01-14-marta/lookup.png){:width="650px" .shadow}

One last thing that reminded me of JetBrains IDEs, was the fact that I could just **start typing to filter items** in the _Recent Locations_ popup menu:

![Filtering in Recent Locations popup](/assets/images/2020-01-14-marta/searching02.png){:width="650px" .shadow}

Great tool! I was even recommending Marta to a colleague of mine, with whom I occasionally share tool recommendations. Then, another week later, while I was looking for the [public repository](https://github.com/marta-file-manager/marta-issues) to raise an [issue](https://github.com/marta-file-manager/marta-issues/issues/623), I found a few information about the author:

![Searching in popups](/assets/images/2020-01-14-marta/yan.png){:width="600px" .shadow}

Okay, this is crazy! 🤯 My initial thought was not too bad. Yan is working in the Kotlin team. The (developer) world is kinda small. 😅

## More Configuration

If you know JetBrains IDEs, you also know how customizable they are. Marta is actually similar in that regards. So my first act was to align all shortcuts with the ones I'm using in Rider. Again, the functionality feels very familiar:

|---------+-------------|
| Action  | Explanation |
|:--------|:------------|
| `core.lookup` | Similar to [Search Everywhere](https://www.jetbrains.com/help/rider/Searching_Everywhere.html) navigation but with additional [prefixes](https://marta.yanex.org/docs/#look-up) |
| `core.actions` | Similar to [Navigate to action](https://www.jetbrains.com/help/rider/Navigating_to_Action.html) |
| `core.favorites` | Similar to the [Favorites window](https://www.jetbrains.com/help/rider/Favorites_Tool_Window.html) whereas entries can be modified via `Edit favorites` action |
| `core.toggle.terminal` | Similar to the [Terminal Emulator](https://www.jetbrains.com/help/rider/Terminal_Emulator.html), which will automatically show/hide depending on the active pane |
| `core.rename` | Similar to the [rename refactoring](https://www.jetbrains.com/help/rider/Refactorings__Rename.html) |

Of course, there are much more options. Like file-system specific actions, or the existential _Dark_ theme. My configuration looks like this:

{% highlight groovy linenos %}
behavior {
    theme "Dark"
}

environment {
    terminal "iTerm"
    textEditor "Visual Studio Code"
}

keyBindings {
    "Shift+Cmd+A" { id "core.actions" }
    "Cmd+Shift+T" { id "core.lookup" }
    
    "Cmd+1" { id "core.favorites" }
    "Cmd+E" { id "core.recent.directories" }
    "Cmd+3" { id "core.toggle.terminal" }
    "Cmd+Shift+3" { id "core.open.terminal.here" }
    "Cmd+4" { id "core.open.editor.here" }
    
    "Cmd+Alt+N" { id "core.new.directory" }
    "Cmd+Shift+N" { id "core.new.file" }
    
    "Cmd+R" { id "core.rename" }
    
    "Return" { id "core.open" }
    "Cmd+Return" { id "core.edit" }
}
{% endhighlight %}

Yan, thank you so much for completing my MacOS developer experience! 👏
