---
layout: post
title: Rapid screenshots in Windows & Linux
category: workflow
date: 2023-03-12 
tags:
  - Windows
  - Linux
  - screenshots
  - file manager
---
Both at work and at home I am heavily using screenshot tools to document findings and highlight meeting content. This short post documents how to assign shortcuts in both windows and Linux for e.g. rapid-fire screenshot taking. Both solutions can also be used for other shortcuts.
<!--more-->
# Windows
At work I frequently use several tools: the file explorer `MultiCommander`, and the screenshot program `greenshot`. With the tool [autohotkey](https://www.autohotkey.com/) we can generate scripts that allow us to reassign keys and map them to launch programs or emit other keys. I mapped my two left mouse side-buttons to take a whole screen screenshot (for rapid fire moments) and to open a selection grab. Furthermore, I reassigned the win+e shortcut to open `MultiCommander` instead of the windows file explorer. The `autohotkey` script looks as follows 
```
XButton1:F11
XButton2:Send ^{PrintScreen}
#e::Run "C:\Users\<username>\Local\MultiCommander (X64)\MultiCommander.exe" "<path_to_startup_folder_left_pane>" "<path_to_startup_folder_right_pane"
return
```
# Linux
The `flameshot` screenshot utility is just amazing. In Linux we can use `xev` to find the name of specific keys (also mouse-buttons). For me the two left side-buttons of the mouse are button8 and button9. `Autohotkey` is not available in Linux, but usually your window manager allows you to bind keys to specific actions. I am using the window manager i3 that also supports [mousebindings](https://i3wm.org/docs/userguide.html#mousebindings)
and to run scripts upon a keypress. We can just add either of these two lines to our i3 config (usually under `~/.config/i3/config`). I am still undecided whether I actually like the pure keyboard shortcut more than using the mouse-button - time will tell.
```
bindsym $mod+g exec xdotool key --clearmodifiers Print
bindsym --whole-window button8 exec xdotool key --clearmodifiers Print
```
`xdotool` helps us to emit key events, what helps greatly as my home keyboard is missing the num-pad and also the printkey. Print is at an inconvenient location anyways...
# Additional Resources
[stackoverflow - keypresses with i3 bindsym and xdotool](https://stackoverflow.com/questions/61272019/infinite-loop-of-keypresses-with-i3-bindsym-and-xdotool)
