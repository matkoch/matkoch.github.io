---
title:		    SVG to XAML
layout:       post
tag:          ['xaml', 'svg', 'resharper', 'testlinker']
description:  Converting SVG to XAML Image with DrawingBrush for ReSharper plugin
---

Unfortunately, the [ReSharpers DevGuide](https://www.jetbrains.com/help/resharper/sdk/README.html) was kind of incomplete for me when it came to [generating compiled XAML icons](https://www.jetbrains.com/help/resharper/sdk/Platform/Shell/Icons/CreatingCompiledIcons.html?search=icon#xaml-icons) for my new [TestLinker](https://github.com/matkoch/TestLinker) plugin. My problem was that I needed a **XAML image with DrawingBrush**, but all I could gather from [iconfinder.net](http://iconfinder.net) was SVG, PNG, ICO and some other format-wise unusefulness. Luckily, things can be converted, although this one needs two tools and a dirty extension renaming.

1. Open SVG with [InkScape](https://inkscape.org/en/)
1. Draw a magenta-filled (`#ff00ff`) rectangle around the existing icon
1. Export as PDF
1. Rename extension to AI
1. Open AI with [ExpressionDesign](https://www.microsoft.com/en-us/download/details.aspx?id=36180)
1. Make the icon 96dpi (File, Document Size...)
1. Resize if necessary
1. Export with format *XAML WPF Resource Dictionary*


After that you can continue as described in the [DevGuide](https://www.jetbrains.com/help/resharper/sdk/Platform/Shell/Icons/CreatingCompiledIcons.html?search=icon#xaml-icons). For illustration, here is the [related commit](https://github.com/matkoch/TestLinker/commit/013717b03a6031a7fc464eac6baa3fb43adb4645) with all the modifications I made.

![TestLinker Options Page]({{ site.url }}/assets/images/svg-to-xaml-options-page.png)

Now that's a shiny happy compiled icon :grin:

