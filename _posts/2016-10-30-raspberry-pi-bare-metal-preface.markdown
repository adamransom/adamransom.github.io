---
layout:     post
title:      Raspberry Pi 3 Bare Metal - Preface
---
The first step with any new language or platform is to write the ubiquitous "Hello, World" example. The equivalent of printing `Hello, World!` in embedded world seems to be turning on an LED, so that is our first challenge!

Before we get into the implementation of that however, I just thought it would be interesting to show the process of getting there.

## Booting Up

In order to grasp an understanding of the inner workings of the Raspberry Pi, I read through various blogs and documents, some of which I linked to in the previous post. At first, I simply wanted to know how the Raspberry Pi boots up and even manages to run our code. After a few—well many—hours of reading, I had a rough idea of the boot process:

1. The Raspberry Pi powers on and at this point the ARM core is **off** and the GPU core is **on**.
2. The GPU runs the first bootloader, which is held in ROM on the SoC (System on Chip). This is similar to the BIOS in conventional PCs.
3. This bootloader reads the SD card and loads the second bootloader from a file named `bootloader.bin`.
4. The second bootloader reads the GPU firmware, also from the SD card, named `start.elf`.
5. Finally, and most importantly for us, `start.elf` reads `kernel.img` and allows the ARM core to execute it.

This final file, `kernel.img`, is the only[^1] one which we provide. It doesn't have to be an actual kernel (and it won't be for a while), this is just the naming convention.

## Shining Lights

After understanding how the Raspberry Pi boots, I then needed to figure out what to put in `kernel.img`.

There were some [excellent][1] [resources][2] [available][3] that outlined the process of turning on the activity LED for the Raspberry Pi 1 & 2. After a few more hours reading I understood the following:

1. The LED is wired to the 16th GPIO pin.
2. We need to tell the GPIO controller to select that pin and then turn it on.

I knew that the Raspberry Pi 3 might have a different pin setup than the previous models and due to limited documentation for the RPi 3, I checked the [Raspberry Pi 3 Schematic][4] diagrams.

Hmm...that's interesting...there is no mention of any LEDs being wired to the GPIO pins. Actually, if we look closely, the ACT LED is in a whole section by itself in the top corner!

That was when I realised this first step might be larger than I anticipated.

## Raspberry Pi 2 and 3 Differences

So, some more googling around was necessary. I discovered that the PWR and ACT LEDs had in fact [been moved off the GPIO to make room for BlueTooth and WiFI][5] and into a GPIO expander, so you could no longer prod the GPIO pins to control the ACT LED.

After yet more googling, I found out that it [may be possible to control these LEDs][6] using the GPU's [mailbox feature][7]. To quote the RPi firmware documentation:

> Mailboxes facilitate communication between the ARM and the VideoCore.

Since the GPU now controls the LEDs on the RPi 3, we need to use the mailbox interface to send a message from the ARM core to the VideoCore (GPU) and ask very nicely if we can control the LEDs.

Whilst the [list of mailboxes][8] specifies there is an LED channel, this wasn't documented at all and it seemed like we were meant to use the [Property tags (ARM -> VC)][9] channel instead. [This post][6] mentioned the existence of two undocumented property tags: `SET_GPIO_STATE` and `GET_GPIO_STATE`, which could be used to control the LEDs.

It felt like I was starting to get somewhere.

## Sending Mail

I was getting closer to understanding what I needed to do in order to make the ACT LED turn on. The [mailbox documentation][7] included good instructions on how to send and receive mail, with sample C code, and it seemed fairly simple. The previously mentioned [forum post][6] also included the format of the message to send to the mailbox with the `SET_GPIO_STATE` tag. There was one more piece to the puzzle and that was knowing where, _in memory_, I needed to send this message.

A [previous blog][1] I read included a nice introduction to memory mapping between ARM physical addresses and VideoCore bus addresses, but this was only for the RPi 2. I needed the mapping for the RPi 3, and there was not yet any documentation on the new chip used in the RPi 3.

In the same manner as the author of that blog found the new mapping for the RPi 2, I checked the [source code][10] of the U-Boot project and discovered that the mapping hadn't actually changed for the RPi 3. Phew!

So, to sum up all the information I just blasted through:

1. The ACT LED is now on the GPIO expander, which is controlled by the GPU.
2. In order to control pins on the GPIO expander, you need to send a message to the GPU via the [mailbox interface][7].
3. The mailbox base memory address is `0x3f00b880`.
4. The tag for the property I need to send to the mailbox is [`SET_GPIO_STATE`(`0x00038041`)][6].
5. The format of the message is the GPIO pin number (`130` for the ACT LED) followed by the state (`1` for on).

So, that leaves us with enough information to actually implement everything and finally turn on that pesky ACT LED.

This post is already long enough and I want to have the implementation in a separate post to make it easier for people to find and read if they simply want to know how to turn on the ACT LED for the RPi 3, so see you in [Part 1][].

[1]: http://www.valvers.com/open-software/raspberry-pi/step01-bare-metal-programming-in-cpt1/
[2]: http://blog.thiago.me/raspberry-pi-bare-metal-programming-with-rust/
[3]: http://www.cl.cam.ac.uk/projects/raspberrypi/tutorials/os/ok01.html
[4]: https://www.raspberrypi.org/documentation/hardware/raspberrypi/schematics/RPI-3B-V1_2-SCHEMATIC-REDUCED.pdf
[5]: https://github.com/raspberrypi/linux/issues/1332#issuecomment-194353863
[6]: https://www.raspberrypi.org/forums/viewtopic.php?f=43&t=109137&start=100#p990152
[7]: https://github.com/raspberrypi/firmware/wiki/Accessing-mailboxes
[8]: https://github.com/raspberrypi/firmware/wiki/Mailboxes
[9]: https://github.com/raspberrypi/firmware/wiki/Mailbox-property-interface
[10]: https://github.com/RobertCNelson/u-boot/blob/8cbb389bb3da80cbf8911f8386cbff92c6a78afe/arch/arm/mach-bcm283x/include/mach/mbox.h#L42

---

[^1]: `bootloader.bin` and `start.elf` are both closed-source and provided pre-compiled by Broadcom.