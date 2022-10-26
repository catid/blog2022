+++
title = "Quest Pro running Debian Linux and Visual Studio Code"
date = "2022-10-26"
cover = "posts/quest_pro_termux/quest_pro_termux.png"
description = "Interesting demo, but needs better support from Meta for real productive use."
+++

## Video Demo

https://www.youtube.com/watch?v=9D6UsUBzAp4


## Recommended Input Devices

I'd recommend turning on Hand Tracking so you do not need to use your controllers to accept prompts.

Either hand tracking or the controllers are required, because keyboard/mouse is not well supported yet so popups cannot be accepted via mouse for example.

Inside the headset, the Settings app has a Devices section where you can pair Bluetooth devices.

You can turn on Pass-Through for the Home environment, so that you can see the real world behind the virtual monitors.  The trick is to click on the clock/battery/wifi button on the task bar, and click the "face mask with lines coming off it" icon.


## Enable Developer Mode

In the Meta Quest app on your phone, select the menu-burger in the lower right.
Select Devices.
Select Meta Quest Pro.  Click [Connect] if needed, sometimes it takes 10 seconds.  I had to reboot the Quest Pro to get it to connect.
Select Developer Mode.
Set it to enabled.

Plug in the headset to your computer with a USB-C cable.  In the headset click the box to Allow the computer to access everything.  You don't need the controllers for this - I just did it with hand tracking turned on.  You'll need to do this each time you replug the headset.


## Install ADB

Download and unzip SDK Platform-Tools from https://developer.android.com/studio/releases/platform-tools

Open a terminal in the unzipped folder.  On Windows you can right click the window showing the extracted folder, and select "Open in Terminal."

Check that the Quest Pro is found by running `./adb devices`:

```
PS C:\Users\mrcat\Downloads\platform-tools_r33.0.3-windows\platform-tools> ./adb devices
List of devices attached
230YC01D8X02T9  device
```


# Download Termux

Download the APK from:
https://f-droid.org/en/packages/com.termux/

Click the `Download APK` link, which was this file: https://f-droid.org/repo/com.termux_118.apk

Save the .apk file in the same folder where you unzipped the Android SDK Platform-Tools.

Back in the terminal, install the APK by running `./adb install ./com.termux_118.apk`.  This takes about 2 minutes:

```
PS C:\Users\mrcat\Downloads\platform-tools_r33.0.3-windows\platform-tools> ./adb install .\com.termux_118.apk
Performing Streamed Install
Success
```

We are now done with the desktop computer, so you can unplug your Quest Pro from the PC.


## Launch Termux

In the Quest app list, select the Sources drop-down in the upper right and select `Unknown Sources...`

Click on Termux to launch it.

It will show a virtual keyboard.  The fast way to dismiss it is to start typing on your Bluetooth keyboard, which will prevent it from popping up again.


## Quest Multitasking Setup

With your Bluetooth mouse you can select the white line at the bottom of the app to move it away to a comfortable distance.

You can hover the mouse over the corner of the window to rescale the window.  I like to make my windows very Tall.

To do multi-tasking, click and hold one of the task bar buttons, and drag it next to Termux.

You should be able to get the Oculus Browser running next to Termux, which is a good setup for what we're going to do next.


## Install Debian and Code-Server

We will use the port from https://coder.com/docs/code-server/latest/termux

You can open this in the browser in your headset right next to the Termux window so you do not need to use your phone or computer.

Just follow the instructions - You have to enter in some commands into Termux to set up the environment as a full Debian Linux install.

The main problem right now is that you cannot copy/paste text from the browser into Termux, so to install NVM you have to enter the command carefully by hand to download the install.sh script.

I ran `nvm install v16.18.0` to install Node.

I did not need to follow the instructions to edit the bashrc scripts etc.

After code-server was installed, I was able to run it, and I used vim to edit its config file to set the password to something more reasonable.


## Things That Don't Work

Here's my laundry list.  If Meta wants this platform to actually work for software engineers (themselves included), then we need a few things fixed.

Better keyboard/mouse support:

(1) The Windows Key on the keyboard should bring up the task tray.
(2) The bluetooth mouse should be able to accept permission dialogs.
(3) When using a bluetooth keyboard, the virtual keyboard should *never* pop up.  Currently it pops up in every app - super obnoxious.

The goal should be to be able to launch apps and use important system features without needing to reach for controllers or use hand tracking.  I'd prefer to just not have hand tracking enabled since it often times puts distracting blobs on top of what I'm trying to do.

(4) Android apps should show up in the task tray.

Currently they don't go into the 3 recent apps list, and instead you have to go through the Apps list all over again.

(5) We really need to be able to copy text from the Meta browser app and paste it into apps like Termux or a code editor.

Enough said.

(6) Home needs to support app layouts, what Windows calls "virtual desktops."

We need to be able to set up some apps, and be able to return to that layout very quickly.  On Windows, I can press Win+Tab or Ctrl+Win+Arrow to rapidly change between sets of apps.  Right now any time I launch something or go to Settings, my layout gets wrecked and I need to fix it.

If all these things are solved, then software development on a VR headset will feel fairly frictionless and similar to a real desktop.


## Productivity App Ideas

Finally, some app ideas that anyone can work on and sell and I would gladly pay over $50 for:

* VR Panel port of https://tabby.sh - Just being able to remotely log into servers from VR would be amazing.  One of the best features of Tabby.sh is that you can automatically resume a large number of remote sessions each time it launches (having it write `tmux a` automatically for you).

Immersed and Virtual Desktop could both do this pretty quick compared to someone starting from scratch.

* VR Panel port of Windows Subsystem for Linux (WSL).  We need the full, multi-user headless version of Ubuntu 22.04 to run on the headset.  That means just a console no GUI.

Even if it's entirely headless (not even a console), you can shell into it using tabby.sh and use Linux tools from a separate app.  But it would be harder to fix if anything went wrong, so probably a console is required eventually.

An alternative to full VM is a proot environment like Termux with https://github.com/termux/proot-distro - I would prefer this if it can still launch all the same apps, since it would be much more efficient than a VM.

All the "use your headset like a computer" apps like Virtual Desktop or Immersed VR are doing remote access GUI things, which is completely useless in most places you'd use a laptop.  Fixing that would get a lot of people excited since it opens up new ways to use the headset.  What Quest is trying to replicate right now is basically what Android has already when you plug a phone into an external monitor.  To get beyond that, we need to be able to launch full desktop apps.  I believe the engineering effort is too high to just port every single desktop experience like VS Code, though that is the long-term play (think on the order of 10 years).  The next practical step is to just virtualize the PC to leverage the available tools on Linux.

* VR Panel port of an X server and/or virtual WM (window manager).

Even with existing Termux, a good X server for Android doesn't exist right now.  Once we have an X server we can run a full desktop environment, we can run Chrome, Code and other *real* apps, even if they'll run a little slower.  X server performance is a big deal, so part of what's hard about this is making it run fast.

An even better option would be a virtual window manager, which would allow an even better experience.  That would give us native rendering for window decoration and the ability to lay out windows around us instead of being constrained to a single "screen" panel.

* VR Panel port of Visual Studio Code.

Using code-server from a browser is just awful compared to a desktop experience.  The problem is that Code is all about the plugins like the remote extensions, and I'm not sure how easy those are to port.  So this might be higher lift than expected.  A deeper port that explodes each of the panes out separately so it's not just one big "screen" panel would be amazing.
