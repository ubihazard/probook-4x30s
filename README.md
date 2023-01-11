HP ProBook 4530s OpenCore
=========================

This repository contains [OpenCore](https://dortania.github.io/OpenCore-Install-Guide/ "Install guide") [EFI folder](https://github.com/ubihazard/probook-4530s/releases/download/v1.0/EFI.7z "Release") that makes it possible to run [macOS](https://support.apple.com/macos) on old Hewlett-Packard ProBook 4530s series Sandy Bridge laptops. Only models with integrated Intel HD3000 graphics are compatible. Other models, such as with NVIDIA GPUs, will require additional configuration tweaks, including turning the dedicated GPU off.

Although this laptop is very old, macOS works surprisingly well on it with pretty much full compatibility. (In fact, I’m writing this document sitting in front of it right now, on macOS.) I’ve used it only for writing text, browsing web, and watching HD YouTube videos anyway (nothing demanding). Don’t expect running XCode with iOS simulator on it, however. Also handy for managing your iThings.

Most ACPI patches are from legendary [RehabMan](https://github.com/RehabMan "Thanks dude") and were carefully ported from his original Clover config. I just mapped the USB ports and assembled the compatible kexts together.

The recommended macOS version to install is Sierra, followed by Big Sur. (You can install both side-by-side.) Sierra is the only version of macOS where I managed to almost completely resolve the HD3000 graphical artifacts and freeze problem (a well-known issue with these old GPUs on macOS). Visual glitches still happen occasionally, but rarely and seem to go away on their own, and the laptop doesn’t hang / freeze eventually. Big Sur would require [Open Core Legacy Patcher](https://github.com/dortania/OpenCore-Legacy-Patcher "OCLP") (OCLP) to restore legacy graphics acceleration.

You can also attempt installing Monterey if you manage to find and swap in a compatible Wi-Fi card for it. But Monterey is the final macOS release you can possibly install on this laptop, – and this is already stretching it quite too far. Ventura is *not* compatible because it requires AVX2 CPU instructions, which Sandy Bridge lacks support for.

A note on HD3000 issues
-----------------------

Even with 8GB of system RAM the Intel GPU on this laptop model is plagued by random graphical glitches, which manifest in grey blocks suddenly popping around, messed up visuals, and most annoyingly, the system eventually freezes and becomes completely unresponsive (except mouse cursor still moving on the screen).

What worked for me to drastically reduce it is calculating the correct KASLR slide value according to this [guide](https://dortania.github.io/OpenCore-Install-Guide/extras/kaslr-fix.html "KASLR slide guide"). In my case the slide value resulted in **8**, which is set in the `config.plist`:

```xml
<key>boot-args</key>
<string>-no_compat_check amfi_get_out_of_my_way=1 slide=8</string>
```

Your value might be different, so if the system panics and doesn’t boot, try calculating it according to the guide linked above and change it accordingly in the `config.plist`.

This seems to work in Sierra, and the laptop no longer locks up, but sadly doesn’t help Big Sur much.

HD3000 VRAM patch
-----------------

With HD3000 it is possible to change the maximum amount of video ram macOS can allocate from system. The patches are available for macOS Sierra+ and can be enabled via OpenCore’s `config.plist`. Additionally, the `AppleIntelHD3000Graphics` kext `Contents/Info.plist` has to be modified for patch to be complete:

```xml
<key>VRAMOverride</key>
<integer>768</integer>
```

The OC `config.plist` patch doesn’t seem to work on Big Sur with OCLP-installed HD3000 kexts. Instead, the `AppleIntelSNBGraphicsFB` (“Sandy Bridge graphics”) kext has to be [modified directly](https://github.com/ubihazard/macos-scripts/tree/main/Scripts#root-patching "Patch guide") with the help of binary hexadecimal editor. And the VRAM size value in `AppleIntelHD3000Graphics`’s `Info.plist` is also different:

```xml
<key>VRAMOverride</key>
<integer>1536</integer>
```

Don’t forget to [rebuild the kernel cache](https://github.com/ubihazard/macos-scripts/tree/main/Scripts#rebuild-kernel-cache) after you are done patching `AppleIntelHD3000Graphics`.

Brightness adjustment after EDID injection
------------------------------------------

If you inject custom EDID for your internal display you will notice that `Fn` brightness adjustment keys will no longer function and the brightness slider in Big Sur would also show up as disabled in control center area. The reason for this is that in case of custom EDID injection macOS insists on having a particular value in device id field of your EDID. You can use the [edidparser.rb](https://github.com/ubihazard/probook-4530s/releases/download/v1.0/edidparser.rb) script which would extract your current EDID and patch it for brightness keys to work again. This script isn’t my work and is provided as-is.

Configuring trackpad
--------------------

Since your laptop battery is probably long dead, macOS would not recognize it. And without a recognized battery modern versions of macOS also prevent changing trackpad settings (go figure).

In reality the trackpad is fully working, – you just can’t access its settings. In order to configure it you would have to manually edit the binary `.plist` file at `~/Library/Preferences/com.apple.AppleMultitouchTrackpad.plist`.

First, copy it to your desktop and convert it from binary format to XML:

```bash
cd ~/Dekstop
cp ~/Library/Preferences/com.apple.AppleMultitouchTrackpad.plist ./
plutil -convert xml1 com.apple.AppleMultitouchTrackpad.plist
```

Make your edits (make sure the syntax is correct) and convert the XML config back into binary format:

```bash
nano com.apple.AppleMultitouchTrackpad.plist
plutil -convert binary1 com.apple.AppleMultitouchTrackpad.plist
```

Finally, replace the original file with your edited copy:

```bash
cp com.apple.AppleMultitouchTrackpad.plist ~/Library/Preferences/
```

*You must log out and log in back for changes to apply.* A reboot is not needed.

A pre-made trackpad configuration file with tap to click is [provided](Library/Preferences/com.apple.AppleMultitouchTrackpad.plist "Trackpad config") and should suit most users. Copy it to `~/Library/Preferences` replacing the original.

Separate config for installer
-----------------------------

For USB installer you would need a different OpenCore `config.plist` modified specifically for installing macOS. It disables some kexts which are useless during installation (Wi-Fi, bluetooth, card reader, etc.), doesn’t modify SIP flags, enables verbose boot (with kernel text messages), and has a different SMBIOS mac model identifier which allows to install Big Sur.

Grab it [directly](config.plist "USB installer OpenCode config") from this repository (not from releases page).

Post-install
------------

Don’t forget to fill in your own SMBIOS information (board serial, system UUID, etc.) after successful installation.

Follow a [guide](https://github.com/Marcuriee/Hackintosh-Guide/blob/main/configuring-smbios.md "Generate SMBIOS") to make a correct SMBIOS for your hackintosh laptop.

Fixing transparency effects in Big Sur
--------------------------------------

In Big Sur (and later) transparent effects, such as blur, render incorrectly on non-Metal GPUs (HD3000 is not capable of Metal API). Luckily, a fix is available, forcing rendering of such effects using legacy method, which doesn’t require Metal support:

```bash
defaults write -g Moraea_BlurBeta -bool true
```

You might also experiment with different blur strength `ASB_BlurOverride`, style of dark window borders `Moraea_RimBeta`, and dark menu bar text `Moraea_DarkMenuBar`:

```bash
defaults write -g ASB_BlurOverride -float 30
defaults write -g Moraea_RimBeta -bool false
defaults write -g Moraea_DarkMenuBar -bool false
```

Clover fallback
---------------

The provided EFI folder also contains a copy of Clover bootloader in case if you have trouble getting OpenCore to work. The included Clover version has only been tested with macOS Sierra and High Sierra, and it definitely won’t be able to boot Big Sur.

Note, however, that Clover and OpenCore don’t mix well together. In my experience, a NVRAM reset is required when switching from Clover to OpenCore, at least in case of High Sierra, or the kernel would panic with weird error message. The NVRAM reset can be performed from OpenCore boot screen: press space if you don’t see its menu entry.

Credits
-------

All credits go to [Acidanthera team](https://github.com/acidanthera), [RehabMan](https://github.com/RehabMan), [dortania](https://github.com/dortania), and the rest of talented individuals who worked hard to make running macOS on regular PCs and unsupported hardware a reality.

⭐ Support
---------

If you find anything of this useful, you can [buy me a ☕](https://www.buymeacoffee.com/ubihazard "Donate")!
