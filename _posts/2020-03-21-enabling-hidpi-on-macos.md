---
title: Enabling HiDPI on MacOS
key: enabling-hidpi-on-macos
tags:
- MacOS
- Notes
mode: immersive
header:
  theme: dark
article_header:
  type: overlay
  theme: dark
  background_color: '#203028'
  background_image:
    gradient: 'linear-gradient(135deg, rgba(34, 0, 170 , .2), rgba(139, 34, 139, .2))'
    src: assets/images/2020-03-21-enabling-hidpi-on-macos/cover.jpg
---

***Disclaimer: Instructions here are provided without any warranty. It's a collection of steps. Also, I've only been able to enable a couple of HiDPI resolutions, but not all that I anticipated to.***

This is mostly a post to remind myself of the trouble I've been going through to **enable HiDPI on an external display**. My HP Envy 27 was a problem child from the very beginning. Apparently, it's [not supported in MacOS environments](https://h30434.www3.hp.com/t5/Desktop-Video-Display-and-Touch/Envy-27s-monitor-won-t-give-full-res/td-p/6008349), which I only found out later. Who would have thought that a monitor can have system requirements? Insane! 🙄 There were a couple of suggested solutions though: using different cables and ports (HDMI 1, HDMI 2, DVI), and rebooting MacOS with and without the display attached. At some point it just _worked_, and I could enable a 1440x900 HiDPI resolution with [RDM](https://github.com/avibrazil/RDM/) (Retina Display Menu). HiDPI resolutions are indicated with a little lightning icon:

![Viewing DSL from UI wizard](/assets/images/2020-03-21-enabling-hidpi-on-macos/rdm.png){:width="600px" .shadow}

Then I wanted to **start streaming on Youtube**, which recommends a resolution of `1920x1080`. Most of my supported resolutions had an aspect ratio of 16:10. Only `1280x720` was 16:9. I already knew that one tool that allows adding _custom resolutions_ on MacOS. However, after giving it another try to add `1920x1080`, all of my HiDPI resolutions stopped working. **Never change a running system, I guess!.**

## Forcing HiDPI resolutions

It seems that Apple makes it particularly hard to make any changes beyond their beliefs. Display resolutions are no exceptions, and are guarded by their [System Integrity Protection](https://support.apple.com/en-us/HT204899). Now here is a step-by-step instruction how to add custom resolutions:

### Disable Apple's SIP

Turn of your Mac, reboot while holding down <kbd>Command+R</kbd>. Wait for the recovery mode to finish loading, then open **Utilities &#124; Terminal**:

![Viewing DSL from UI wizard](/assets/images/2020-03-21-enabling-hidpi-on-macos/terminal-in-recovery-mode2.jpg){:width="600px" .shadow}

Afterwards, enter the two commands `csrutil disable` – which should be confirmed by a message – followed by `reboot`.

### Mount System Folder

Since MacOS Catalina, we additionally need to mount the `/System` as being writable. The command of choice is `sudo mount -uw /`. Be assured that there is another reboot planned at the end of the instructions.

### Determine Display IDs

For the next step, we need to determine the `DisplayVendorID` and `DisplayProductID` of our display. The command `ioreg -lw0 | grep IODisplayPrefsKey` will give a list of all the _active_ devices:

```bash
"IODisplayPrefsKey" = "IOService:/AppleACPIPlatformExpert/.../AppleBacklightDisplay-610-a03e"
"IODisplayPrefsKey" = "IOService:/AppleACPIPlatformExpert/.../AppleDisplay-220e-3417"
```

As far as I know, `AppleBacklightDisplay` usually denotes the notebook's own display. The postfix can be read as `-<VendorID>-<ProductID>`. In my case, the vendor ID is `220e` and the product ID is `3417`.

### Add System Overrides

[Enable-HiDPI-OSX](https://github.com/syscl/Enable-HiDPI-OSX) is a project hosted on GitHub that allows adding overrides in the most convenient way I've seen. There's also [another website](https://comsysto.github.io/Display-Override-PropertyList-File-Parser-and-Generator-with-HiDPI-Support-For-Scaled-Resolutions/), but for me it didn't really work well.

First we need to download the `enable-HiDPI` and `restore` script:

```bash
curl -o ~/enable-HiDPI.sh https://raw.githubusercontent.com/syscl/Enable-HiDPI-OSX/master/enable-HiDPI.sh
curl -o ~/restore https://raw.githubusercontent.com/syscl/Enable-HiDPI-OSX/master/restore

chmod +x ~/enable-HiDPI.sh
chmod +x ~/restore
```

Now gather up all your courage and execute `sudo ./enable-HiDPI.sh`, which initially asks us to select the display we want to change resolutions for:

```bash
         Table of monitors
------------------------------------
  Index  |  VendorID  |  ProductID
------------------------------------
    1    |    0610    |    3ea0
    2    |    220e    |    1734
------------------------------------
Choose the display to enable HiDPI[Exit/1/2]:
```

After making our selection, we need to provide the resolutions we want to add. Unfortunately, the interface is not really intuitive, but we need to specify the resolutions one-by-one:

```bash
Enter the HiDPI resolution (e.g. 1600x900, 1440x910, ...), 0 to quit: 1920x1080
Enter the HiDPI resolution (e.g. 1600x900, 1440x910, ...), 0 to quit: 1440x900
Enter the HiDPI resolution (e.g. 1600x900, 1440x910, ...), 0 to quit: 1280x720
Enter the HiDPI resolution (e.g. 1600x900, 1440x910, ...), 0 to quit: 0
[  OK  ]  Backup /System/Library/Displays/Contents/Resources/Overrides.
[  OK  ]  Done. Reboot then use Retina Display Menu (RDM) to select the HiDPI resolution just injected!.
```

In my case, the applied changes are saved to `/System/Library/Displays/Contents/Resources/Overrides/DisplayVendorID-220e/DisplayProductID-3417`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
	<key>DisplayProductID</key>
	<integer>13335</integer>
	<key>DisplayVendorID</key>
	<integer>8718</integer>
	<key>scale-resolutions</key>
	<array>
		<data>AAAPAA==</data>
		<data>AAALQA==</data>
		<data>AAAKAA==</data>
	</array>
</dict>
</plist>
```

There's an article describing [how those values are encoded](https://comsystoreply.de/blog-post/force-hidpi-resolutions-for-dell-u2515h-monitor), but for me they seem to be a bit too short. Whatever! I also copied the `scale-resolution` entries to my MacBooks own override file, to allow setting the same resolution on both.

### Side-Effects

While changing the override files back and forth, it seemed that even though the `1920x1080` resolution doesn't work, it had to be added to make other resolutions work. I have absolutely no explanation for that, but at this point I'm just happy to have a few HiDPI resolutions back.

## Conclusion

For the YouTube streaming I'm now using a **display resolution of `1280x720` with recording resolution of `1920x1080`** for the output. Anyways, this seems to be a much better choice, since I don't need to change font sizes in every application.
