---
title:        Essential Debugging with VisualStudio
layout:       post
tag:          ['productivity', 'essentials', 'debugging', 'visual studio']
description:  Essential features when debugging with VisualStudio
unpublished:  true
---

In this post we will look at the most basic features, which are indispensable while debugging in VisualStudio.

## Shortcuts

Many of the shortcuts I talk about, have their corresponding toolbar button. However, for increased productivity, this small set of shortcuts should be learned by everyone.

Most developers are already familiar with some basic shortcuts. The most important once for **starting the application** are:

- `Debug.Start`: default on <kbd>F5</kbd>
- `Debug.StartWithoutDebugging`: default on <kbd>Ctrl</kbd>+<kbd>F5</kbd>

Although I really wished that those two shortcuts also had a default:

- `Debug.DetachAll`: manually assigned to <kbd>Ctrl</kbd>+<kbd>Shift</kbd>+<kbd>F7</kbd>
- `Debug.TerminateAll`: manually assigned to <kbd>Ctrl</kbd>+<kbd>Shift</kbd>+<kbd>F8</kbd>

Some other basic ones for **navigating through the stack trace** are:

- `Debug.StepOver`: default on <kbd>F10</kbd>
- `Debug.StepInto`: default on <kbd>F11</kbd>
- `Debug.StepOut`: default on <kbd>Shift</kbd>+<kbd>F11</kbd>
- `Debug.ToggleBreakpoint`: default on <kbd>F9</kbd>

A more advanced feature is to select the next executed statement either by using the `Debug.SetNextStatement` action, whose default shortcut is on <kbd>Ctrl</kbd>+<kbd>Shift</kbd>+<kbd>F10</kbd>, or by dragging the yellow indicator in the sidebar:

GIF

## Exceptions Dialog

First Chance
Just My Code?

## Modules ??

## Parallel Stacks ??

Freeze

## Lazy Debugging

Debugger.Launch();
Attach to Process Alt+D,P (OzCode?)