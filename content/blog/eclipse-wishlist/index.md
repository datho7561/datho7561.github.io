---
title: "Eclipse Wishlist"
date: 2025-08-14
comments: true
---

![A poorly executed, pixelated drawing of the Eclipse logo](./eclipse-logo.png)

The Eclipse IDE has a lot to catch up on if it ever wants to compete with the likes of
VS Code, Intellij, [Zed](https://zed.dev/), [Neovim](https://neovim.io/), and [Helix](https://helix-editor.com/).
There are not enough people motivated to work on Eclipse
and too much internal politics adverse to change to ever address these pain points.
The last two entries I included might even be too difficult to ever complete.
However, I can dream.
What follows is my wishlist.

## Fix drag and drop to create split views

Looking at two files at once is an essential feature of any text editor program or IDE.
In Eclipse, you're supposed to be able to drag and drop a tab to split the editor into two views,
but on Linux at least, it usually doesn't work.
Sometimes it works fine, but usually title of the tab becomes underlined and you can't move it at all.
Even worse, it sometimes pops the tab out to it's own separate window instead, which is _very_ annoying.
Given how unreliable the feature is, I often just give up on using it.
In other programs like VS Code and Neovim, it's very simple to do this.

## Theming

It's possible to change the colours of the Java text editor to fit a colour scheme.
However, you have to manually change each colour one by one in a clunky UI.
You could also manually edit one of the settings files to set these, but that's also annoying.
Either way, this is a far cry from other IDEs,
which often come prepackaged with > 10 different themes that you can switch between using a drop down menu.

To add insult to injury, these settings only apply to the Java editor.
If you open an XML or YAML file, it uses a different set of settings to control those colours,
so you will also need to fix those in a different part of the UI.

Also, these settings are per-workspace.
If you use multiple workspaces, you need to copy/paste the settings into new workspaces you create.
This problem is exacerbated by the fact that the correct settings files are not easy to find on your filesystem,
nor does Eclipse provide an easy way to edit them from within the editor.

Furthermore, it's very difficult to share your themes.
[I have settings for a gruvbox theme in a gist](https://gist.github.com/datho7561/c67525f7ed31bff911ab03bdd6821ad2),
but someone hoping to use them would need to copy these into those hard to find settings files
and make sure there's no duplicate entries in those files.
There used to be an "Eclipse themes" plugin to manage and distribute themes,
but it was abandoned, likely because it was too complex to maintain.
No one's developed a successor to it.

Finally, it's very difficult to theme the rest of the UI outside of the textboxes.
From my understanding, it involves GTK4 theming on Linux.
It's so complex that it disuaded me from even attempting it.

Almost all other IDEs provide proper theming,
where you specify the name of the theme and it applies all the colours instantly.
All other IDEs make it easy to use a theme that someone else wrote,
and many provide tools for writing your own themes and distributing them to others.

## Improve the version control gutter and enable it by default

When working with code checked out from a version control system,
Eclipse has a feature adds a marker on the left margin of the text editor to show the unstaged changes.
For some inexplicable reason, it's disabled by default.
Every other IDE has this enabled by default; it's a basic feature at this point.
It takes a solid 5 minutes of digging around to find the setting to enable it in Eclipse if you've never done it before.
Also, the marker covers the line numbers instead of appearing to the left of them in it's own column,
which is a baffling decision given every other editor/IDE has converged on using a separate column.
Furthermore, some IDEs like VS Code let you click on the column with the marker to stage or revert the relevant hunk;
Eclipse doesn't have anything like this.
Seeing what changes you've made since the last commit is an indispensible
IDE feature when working on small changes to an established codebase,
and it's baffling that Eclipse's implementation is disabled by default and not that good.

## Whenever possible, decouple features from UI

Despite the outward appearance of being over-engineered,
Eclipse doesn't have a clean separation between UI and non-UI code.
The main example I'm thinking of is that the Java quickfixes have historically been
tightly coupled with the UI components.
One of my collegues, Rob Stryker, has worked really hard to address this by slowly decoupling this code from the UI,
so that the quickfixes can be used in jdt-ls, the Java language server based on Eclipse.
(jdt-ls is a background process that runs without a UI of it's own).
Rob accomplished quite a lot through his changes,
however this decoupled code still lives in
[a git repository explicitly labelled "UI"](https://github.com/eclipse-jdt/eclipse.jdt.ui/tree/master/org.eclipse.jdt.core.manipulation).
Also, I believe there are other cases where there's no clean separation yet.

## Document and unify the plugin system

Eclipse's plugin system is very powerful in theory,
but it's foreign to many Java developers new to the Eclipse ecosystem.
It uses a system called OSGi, which is very helpful,
since it handles hot loading/unloading of plugins and can handle plugins using different versions of the same dependency.
However, it increases the complexity of developing a plugin.
OSGi is a specification, and there are multiple implementations.
Eclipse provides it's own implementation, which predates the specification,
so even if you've worked with an OSGi project in the past, there might be some differences.
To make onboarding new developers easier,
the plugin system should be well documented and provide clear examples.

The software to compile your plugin code on the command line and software
to compile your plugin from within Eclipse while developing it are two completely different codebases.
This means that sometimes something works in one place and not the other.
For instance, your plugin may compile on the command line,
but you will need to click through a couple UIs
and know to hit certain buttons when you import it into Eclipse,
and it still might not work.

A major limitation of the in-IDE compilation tool, PDE (Plugin Development for Eclipse),
is that you can only use one "target platform" at a time.
The target platform is one of the documents that specifies your plugin's dependencies.
This means that you cannot properly develop two plugins at the same time,
since if you have one of the target platforms set,
then PDE is using those dependencies for both plugins when attempting to compile.

This seems like a very arbitrary limit;
when using Tycho, the command line tool for compiling Eclipse plugins,
the target platform is specified to Tycho in the Maven POM.
PDE should be able to do something similar.

Ideally, I believe these code bases should be combined.
These should not be two separate systems;
compilation should work identically or close to identically in both scenarios.
If for some reason this is impossible, these systems should have the same
feature set and work as similar as possible.

## End Rant

After using and working on Eclipse over the years,
I've become acutely aware of its shortcomings.
I really hope that someday at least some of these get addressed,
but I'm not holding my breath.
It's very difficult to get buy-in from the maintainers to add improvements to Eclipse,
and it's even more difficult to convince people to pay you to work on it.
Unless there are systemic changes at Eclipse and a surge in company investment in the IDE,
Eclipse will continue to lose committers and users.

