---
layout: post-light-feature
title: "What's up with Windows 10?"
description: "Windows 10 brings a lot of good changes, but at least one strange choice."
category: articles
tags: [windows, microsoft]
date: 2015-07-25
image: 
  feature: 2015-07-25/banner.png
---

While working with MaxTo I come across a lot of edge cases. I initially thought [this bug](https://github.com/digitalcreations/MaxTo/issues/50) was an edge case that would be fixed the closer Windows 10 came to RTM, but I was mistaken.

MaxTo relies on being able to line up windows perfectly against the edges of other windows (and your monitor). Imagine my surprise when I noticed that in Windows 10, `MoveWindow(hWnd, 0, 0, 200, 200, true)` does not visually align a window in the top left corner. Neither does `SetWindowPos`. This is strange, I thought, and imagined it was some edge case bug in Windows 10. I tried reaching out to Microsoft (this was four months ago now), but I couldn't get a reply out of them. So I waited until closer to RTM.

Here's what you can do to reproduce the problem (least amount of code).

1. Create a new WinForms project (I did it in C#).
2. Add a button to the form.
3. Create a click handler for that button.
4. Type `Left = 0;`.
5. Run.
6. Click the damn button already.

If you do this on any earlier version of Windows, the window aligns perfectly to the left-hand side of the screen. If you do this on Windows 10, you get this:

![Window apparently aligned 7 pixels from the left-hand side of the screen](/images/2015-07-25/windows10bug.png) 

Yes, that image shows the left-hand side of my monitor. The window is visually aligned 7 pixels from the left. After quite a bit of soul-searching, Spy++-ing and image editing, this is the conclusion: In Windows 10, the window borders are invisible (meaning transparent), but **they are still there**.

You can easily see this for yourself, by opening e.g. Notepad and hovering the mouse just outside what you perceive as the border. You'll see the cursor change to its resizing icon, meaning the window border is there, but it is invisible.

![Window with borders painted in](/images/2015-07-25/windows10bugborder.png)

Since they just made the regular borders invisible, the effect is only visible on the 3 sides of the window where there normally is a border (left, bottom and right). The caption of the window is controlled by Windows, and has its own resizing handle. But it also means that the resizing handle on the top of the window is only 1 pixel tall (it has been that way for a while, actually).

So what does this mean for you as a developer? In most cases, not much. The client area will be exactly the same size on Windows 10 and 8.1 with the same `MoveWindow` call. The only change is in the visual size of the window, and it is only visible if you try to align windows along some other edge.

To align a window to an edge, you simply have to enlarge the rectangle you resize to when resizing. This is what Microsoft has done to get "Aero Snap" (or whatever we should call it now) to work. Note that there are caveats here: 

- If you enlarge it when resizing, you also have to contract it when reading the size from it.
- Windows resizing handles will overlap. So the top-most window will be the one you resize.