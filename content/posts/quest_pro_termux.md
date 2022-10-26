+++
title = "Quest Pro running Debian Linux and Visual Studio Code"
date = "2022-10-26"
cover = "posts/quest_pro_termux/quest_pro_termux.png"
description = "Interesting demo, but needs better support from Meta for real productive use."
+++

## Video of everything working:

https://www.youtube.com/watch?v=9D6UsUBzAp4


## Recommended Input Devices

I'd recommend turning on Hand Tracking so you do not need to use your controllers to accept prompts.

Either hand tracking or the controllers are required, because keyboard/mouse is not well supported yet so popups cannot be accepted via mouse for example.

Inside the headset, the Settings app has a Devices section where you can pair Bluetooth devices.

You can turn on Pass-Through for the Home environment, so that you can see the real world behind the virtual monitors.  The trick is to click on the clock/battery/wifi button on the task bar, and click the "face mask with lines coming off" it icon.


## Enable developer mode:

In the Meta Quest app on your phone, select the menu-burger in the lower right.
Select Devices.
Select Meta Quest Pro.  Click [Connect] if needed, sometimes it takes 10 seconds.  I had to reboot the Quest Pro to get it to connect.
Select Developer Mode.
Set it to enabled.

Plug in the headset to your computer with a USB-C cable.  In the headset click the box to Allow the computer to access everything.  You don't need the controllers for this - I just did it with hand tracking turned on.  You'll need to do this each time you replug the headset.

Install ADB on your computer:

Download and unzip SDK Platform-Tools from https://developer.android.com/studio/releases/platform-tools

Open a terminal in the unzipped folder.  On Windows you can right click the window showing the extracted folder, and select "Open in Terminal."

Check that the Quest Pro is found by running `./adb devices`:

```
PS C:\Users\mrcat\Downloads\platform-tools_r33.0.3-windows\platform-tools> ./adb devices
List of devices attached
230YC01D8X02T9  device
```


# Download Termux:

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


## Launch Termux:

In the Quest app list, select the Sources drop-down in the upper right and select `Unknown Sources...`

Click on Termux to launch it.

It will show a virtual keyboard.  The fast way to dismiss it is to start typing on your Bluetooth keyboard, which will prevent it from popping up again.


## Quest Multitasking Setup:

With your Bluetooth mouse you can select the white line at the bottom of the app to move it away to a comfortable distance.

You can hover the mouse over the corner of the window to rescale the window.  I like to make my windows very Tall.

To do multi-tasking, click and hold one of the task bar buttons, and drag it next to Termux.

You should be able to get the Oculus Browser running next to Termux, which is a good setup for what we're going to do next.


## Install Debian and Code-Server:

We will use the port from https://coder.com/docs/code-server/latest/termux

You can open this in the browser in your headset right next to the Termux window so you do not need to use your phone or computer.

Just follow the instructions - You have to enter in some commands into Termux to set up the environment as a full Debian Linux install.

The main problem right now is that you cannot copy/paste text from the browser into Termux, so to install NVM you have to enter the command carefully by hand to download the install.sh script.

I ran `nvm install v16.18.0` to install Node.

I did not need to follow the instructions to edit the bashrc scripts etc.

After code-server was installed, I was able to run it, and I used vim to edit its config file to set the password to something more reasonable.


## What's next?

Well, I didn't much like code-server for text editing due to lots of bugs and general input sluggishness.

Hopefully someone writes a VR port of VS Code, so we can use that for doing professional software development where we mostly will shell into a build server rather than building things locally.

Also I hope that Quest Pro gets better support for keyboard/mouse.  I would like the Windows key to bring up the app bar so you don't need to use a controller for that.  I would also like to be able to click on permission boxes with the bluetooth mouse.
Too many apps have the virtual keyboard pop up when it's not wanted, so that feels pretty janky when you have a real keyboard in front of you.

Also, I would love to have the concept of desktops like in Windows, which seems like a minimal ask.  On Windows, I usually have about 3-4 desktops full of apps that I switch between, and it's a huge time saver.  On Quest I'd love to be able to set up a few apps on each desktop the way I want, and be able to get in and out of that desktop so I can use other apps and return to the same consistent workspace.

We really need to be able to copy text from the Meta browser app and paste it into apps like Termux or a code editor.
