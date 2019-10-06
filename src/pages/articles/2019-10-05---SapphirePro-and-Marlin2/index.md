---
title: TwoTrees Sapphire Pro and Marlin 2.0
date: "2019-10-05T23:00:00.000Z"
layout: post
draft: false
path: "/posts/sapphire-pro-marlin/"
category: "Project"
tags:
  - "sapphire pro"
  - "marlin"
  - "firmware"
  - "MKS Robin Nano"
description: "The Sapphire Pro from TwoTrees comes shipped with a MKS Robin Nano board. While the board is open, its firmware is not and so you can hardly modify anything. Instead of replacing the board I managed to get Marlin working this machine."
---

## The MKS Robin Nano

The Robin series from MKS features the STM32f103vet6 microprocessor including a FSMC interface. This way the display can directly be driven from the same microprocessor and memory. The typical other TFT screens usually run on their own and are just interfaced via G-Code.

I can just guess the advantages and disadvantages until now, but I always disliked the idea of the standalone displays and also sticked to a classic 128x64 LCD on my other 32bit machine.

### MKS firmware

My Sapphire Pro already came with an updated firmware version (v1.0.3) and had a basic UI with the most common controls. I won't be too harsh with this type of printer control, because I think this the right direction, but we are still not where we need to be. Using this UI feels like the first touch screen phones from the early 2000s, than a current development.

Looking a little bit deeper I found the config file, that allows a decent amount of tweaking. If you're familiar with some printer firmware configuration files, you may recognize something here:

```
(..)
#=============================== Bed Leveling ==============================
>BED_LEVELING_METHOD 		0	# 0:NULL_BED_LEVELING; 3:AUTO_BED_LEVELING_BILINEAR; 5:MESH_BED_LEVELING
>GRID_MAX_POINTS_X 		3	# the number of grid points per dimension. <= 15
>GRID_MAX_POINTS_Y 		3	# the number of grid points per dimension. <= 15
(..)
```

Does this look familiar? Don't get me wrong, I don't want to diminish the work some programmer put into this, but **IF YOU STEAL FROM THE GPL COMMUNITY AT LEAST HIDE IT PROPERLY** or just release the sources.

[MKS Robin Nano on Github](https://github.com/makerbase-mks/MKS-Robin/tree/master/MKS%20Robin%20Nano)

#### a comment on china and the GPL

I recently read a little twitter story from Naomi Wu about this. While in the western (printer) world, we're making our products outstanding and special by creating and controlling our sources for parts and their quality, because it's hard and time consuming here, the chinese can source the parts in the quality they need much faster and easier. To keep them ahead in business they try not to disclose their part-sources, but the software.

I can understand this problem and behavior on one hand, but on the other hand let's be real: most of the chinese printer firmware and software we get currently is default garbage and often outdated. (look at all the old 15.x Cura stuff that's often included). But two of the leading western printer builders, Prusa and Ultimaker make money, because their firmware, software, support and user experience are gold, not their parts.

Has any other printer on the market, that is not a 1:1 Prusa clone "borrowed" and customized Prusas' software for their printers? Not that I'm aware of. It's the opposite, Prusa and other heavy Marlin modders are actually contributing their features back to the main branch to enhance the firmware for everyone, also for the chinese printers.

I don't know what has to happen in the chinese printer market to realize, what open-source not literally, but actually means and how to embrace it, rather than ignore.

I have the feeling, looking at the MKS github repository they are clueless. If we want to make this whole world a better place, we need more personalities directly in china baptising the idea and workflow of open-source, like Naomi does.

[Naomi Wu about sourcing parts on twitter](https://twitter.com/RealSexyCyborg/status/1173759821486493697)

[Naomi Wu speaking at COSCon'18](https://www.youtube.com/watch?v=WFpiHqJB77w)

### Marlin and Robin

This out of the way, we can all agree, that having the latest Marlin build on your printer, together with reasonable controls, would be great **YES** and that the current MKS firmware limits this printer to just the printer it came for under 300 Euros, not a base for more experiments. **BOO**

#### how to get new firmware on the board

Like a lot of the 32 bit boards, this one also has a bootloader, that sucks files from the SD card and updates on start-up, no complicated serial STM-Link clamps. With the current MKS firmware you can also throw config files and encoded pictures at it to update the appearance and change the settings with just one reboot.

Since there was no real instruction manual for doing this and the files from the main branch of marlin wouldn't load I had to research a bit and found the clue: the bootloader looks for *Robin_nano35.bin* and not *Robin_nano.bin* like the compiler spits out. So has been customized by MKS/TwoTrees. (with v1.0.3)

After an evening of digging through github, forums, datasheets, correcting compiler fails and dozens of SD card in-and-outs, I had 2 versions of marlin running on my printer with a decent configuration, that would actually print on the Sapphire Pro:

* the pretty old, customized version directly from the MKS repository:
  * classic marlin interface in the middle of the screen and 3 touch buttons representing the encoder knob
  * unfortunately this version has various bugs and the changes, that were made to get the display and touch control running, were not tracked.
  * updating this version commit for commit or dig in the branches for this exact version before the changes was no option for me.
  * if I want to run this printer with marlin, than not like this
* a current marlin built
  * unfortunately the class, that prints the marlin on the screen was made for a different resolution and only has basic configuration options burried in the code.
  * so the display is too small, stuck in the 0,0 corner, while the touchzones were correctly upscaled. This may work and one could use it like this, but let's be honest: that's not a final solution either.

### old alpine returns

There once was a time in my life far far away, when I was programming for CNC microscopes and did a lot of machine vision stuff. This knowledge clearly helped to realize, what the problem was and that the solution was hast a little hack away.

The *u8g_dev_tft_320x240_upscale_from_128x64* class, that feeds the display, just did what it's called: Upscale the ancient 128x64 pixel UI by 2 and shove it in a framebuffer, that's connected via the FSMC to the TFT. Wow, that's not only elegant, but also really easy to adjust.

I just have to hope, that the whole framebuffer-FSMC-TFT construct would allow higher resolutions, than the hardcoded 320x240, as the 3.5" TFT actually has 480x320 pixels. (I guess this was coded for a smaller 2.8" TFT, that is also distributed by MKS)

With a little pixel math, a design for the bigger resolution was created

```
        <-- 480 px -->
   __________________________
  |_| top offset: 32 px    |_|
  |_|______________________|_|  /\
  | | Marlin display       |_|  |
  | | 384*192 px           |_|  |
  |_|______________________|_|  320 px
  |_| middle spacing 16 px |_|  |
  |_|______________________|_|  |
  | | touch UI: 384*64 px  |_|  \/
  | | element Y: 2+60+2 px |_|
  |_|______________________|_|
  |_| bottom offset: 16 px |_|
  |_|______________________|_|
   ^- left offset:          ^- right offset:
      48 px                    48 px
```

All we need to do now is to upscale by 3 instead of 2 and hope for the best.

Operation successful, now the touch buttons also needed some treatment and voilà!

To get the old vibes going, we can also let it look like an actual RepRap Full Graphic Smart Controller.

### config tweaking

What now had to follow is the typical boring job of adjusting and fine tuning features, values and limits, until it suits the printer.

### How to install it on your Sapphire Pro

This will be covered more in detail in a separate post, but here are all the resources you need.

You'll need platformIO to compile everything. Rename your *Robin_nano.bin* to *Robin_nano35.bin*, put it on your microSD and start the printer with it and it should update.

* [my github branch for the Sapphire Pro](https://github.com/inib/Marlin/tree/2.0.X-SapphirePro-3.5TFT)
* [commits made](https://github.com/inib/Marlin/commits/2.0.X-SapphirePro-3.5TFT)

***please check the commits and if the config suits your printer configuration***
*(eg. if your bowden tube is shorter than mine, the filament change feature may damage your hotend or extruder!)*

### Conclusion

And that's, where the Sapphire Pro is now. With a few evenings of tinkering we have released this printer kit from its MKS chains, just waiting now for all our upgrades and fancy features. Why it had to be delivered with this firmware-chimera abomination, I have no clue.

If someone from any chinese printer manufacturer actually reads this: ask us, ask the community, ask me, if we can help, beta test your stuff, improve and maintain firmware or just give you an idea what features we would like to see - we are willing to help - as long as you are complying to the rules, that come bundled with these pieces of software.

happy printing,
alpine

## related links

* [TwoTrees 3d printers](https://www.twotrees3dprinter.com/)
* [MKS Makerbase](https://www.makerbase.com.cn)
* [Marlin issue on this topic](https://github.com/MarlinFirmware/Marlin/issues/9771)
* [MKS issue on this topic](https://github.com/makerbase-mks/MKS-Robin/issues/124)
* [(German) Megathread Drucktipps3D Forum](https://drucktipps3d.de/forum/topic/schon-gesehen-sapphire-s-pro/)
* [(German) Marlinthread Drucktipps3D Forum](https://drucktipps3d.de/forum/topic/sapphirepro-mit-marlin-2-0-mks-robin-nano/)
