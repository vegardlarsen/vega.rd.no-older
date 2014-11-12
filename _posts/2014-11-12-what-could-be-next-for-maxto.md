---
layout: post-light-feature
title: "What could be next for MaxTo"
description: "Maxto has looked much the same lately. Here's what might be coming."
category: articles
tags: [maxto]
date: 2014-11-12
image: 
  feature: maxto.jpg
---

Up until a week ago, the latest update for MaxTo was over 3 years old. The reasons for this were many, but one of the main issues holding it back was an internal restructuring that didn't go exactly as planned. This restructuring ties nicely into what my long-term goals are for MaxTo, which I'll be talking about in this post.

*TL;DR*: There are images at the bottom.

## How it started out

MaxTo really started out as a weekend project for me, as many of my projects do. In the beginning MaxTo only supported 32-bit programs, because the computer I had only supported 32-bit programs. 

<figure>
	<img src="/images/maxto/structure-v1.png">
	<figcaption>MaxTo's initial organization was very simple.</figcaption>
</figure>

As you can see, this structure was nice and simple. MaxTo's main process made the call into Windows to load the hook DLL into every process in the system. The hook then reported back to the main process using window messages. Easy as pie.

## Enter 32 more bits

When we implemented 64-bit, it was done as a "quick fix". If you want to maintain a software project over time, steer clear of these. There is a restriction that a 32-bit process cannot set a 64-bit hook &mdash; which makes sense. So we made a 64-bit helper process that MaxTo started, and communicated with that instead of directly with the hook DLL.

<figure>
	<img src="/images/maxto/structure-v2.png">
	<figcaption>The organization suddenly got more complex.</figcaption>
</figure>

The problem with this was that some things could not easily be communicated back to MaxTo's main process anymore, and certain messages had to go through the helper process. So now we had two communication channels for 64-bit, and one channel for 32-bit. This took quite a bit of coordination to pull off. This design violates the DRY principle, since there is a lot of duplicate code.

## The 3-year refactor

To get around this blatant violation of the DRY principle, I decided to refactor MaxTo in two stages. This is the first stage. 

<figure>
	<img src="/images/maxto/structure-v3.png">
	<figcaption>Going symmetric for the first time.</figcaption>
</figure>

To be fair to myself, I didn't really spend 3 full years on this. Most of the refactoring was done in 2012, but after the refactoring half the functionality was left broken in subtle ways. So it took us a while.

In this latest release, there are two helper processes running in the background, one per bitness. The hook DLLs are, as before, compiled from the same source.

## What isn't working now

There are quite a few things that is bothering me regarding the current MaxTo:

- UI code is mixed with core "business" logic. This is a big one, which makes it hard to change anything without affecting other parts. This needs to be fixed.
- There is still quite a bit of overcommunication. Some messages are sent back and forth between processes to determine if a window can be maximized or not. There is at least one known bug caused by this at the moment.
- A lot of the communication over window messages is very ineffective, and takes a lot of coordination to get right.
- The helper process is written in native code, making it very hard to read settings directly. Ideally, the helper process would know what to do with messages, and not have to send it to the core.

## What's next

Part two of my long-term plan to reduce our technical debt, is to introduce a core message bus and extracting the user interface into a separate process. Introducing the message bus requires the helper processes to be rewritten in managed code, but most of that code can be extracted from the current code.

<figure>
	<img src="/images/maxto/structure-v4.png">
	<figcaption>Getting closer to a schematic drawing of the USS Enterprise.</figcaption>
</figure>

In this design, there are a few new items:

- A separate UI process, which will only run when it is needed. As such, MaxTo will run headless most of the time, and start the UI process when it is needed.
- The core will have only three capabilities: 
  1. Message hub for passing messages through the application.
  2. Ensuring that the helper processes are running.
  3. Ensuring that the UI process is running when it is needed.
- This means the helper processes take over a lot of the responsibility. They will now read the settings by themselves, know where the regions are, and be able to control things much more closely.

Communication between the helper process and the injected hook DLL will still happen over window messages, but only between those two processes. This ensures that the responses to events will not be slower than they are today &mdash; they may even be faster.

## What will this look like?

I know, most people only cares about what things look like, not how they work. So, I have prepared a few mockups of the design I'll be aiming for. It may not end up exactly like these mockups, but hopefully it will be close.

The design is inspired by the Modern UI design language, but purposefully held back to work well on the desktop.

### Options

<figure>
	<img src="/images/maxto/design_options_hotkeys.png">
	<figcaption>The hotkeys section of the options dialog.</figcaption>
</figure>

<figure>
	<img src="/images/maxto/design_options_transparency.png">
	<figcaption>There will be a slider next to the illustration to let you set the transparency.</figcaption>
</figure>

### Region setup

<figure>
	<img src="/images/maxto/design_main.png">
	<figcaption>Rough draft.</figcaption>
</figure>

<figure>
	<img src="/images/maxto/design_main_join.png">
	<figcaption>When you hover close to a border, the join button becomes visible.</figcaption>
</figure>

<figure>
	<img src="/images/maxto/design_main_split.png">
	<figcaption>Hovering over a split button shows a preview of what will happen.</figcaption>
</figure>

## Summary

Work on this enhanced version of MaxTo will start now. If there are any bug fixes needed for the current release, they will be released in the mean time. I am very open to suggestions as to what to do with MaxTo.

Just a few days ago I opened a [issue tracker for MaxTo](https://github.com/digitalcreations/MaxTo/issues) so everyone can see the work on MaxTo as it happens. I am hoping to post updates here as this refactoring progresses, but I make no promises.

*[DRY]: Don't Repeat Yourself
*[WPF]: Windows Presentation Foundation