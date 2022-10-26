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

(4) We really need to be able to copy text from the Meta browser app and paste it into apps like Termux or a code editor.

Enough said.

(5) Home needs to support app layouts, what Windows calls "virtual desktops."

We need to be able to set up some apps, and be able to return to that layout very quickly.  On Windows, I can press Win+Tab or Ctrl+Win+Arrow to rapidly change between sets of apps.  Right now any time I launch something or go to Settings, my layout gets wrecked and I need to fix it.

(6) We need a VR port of Visual Studio Code.  Using code-server from a browser is just awful compared to a desktop experience.

If all these things are solved, then software development on a VR headset will feel fairly frictionless and similar to a real desktop.
