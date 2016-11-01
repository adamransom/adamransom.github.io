---
layout:     post
title:      Raspberry Pi 3 Bare Metal - Part 1
---
The first step with any new language or platform is to write the ubiquitous "Hello, World" example. The equivalent of printing `Hello, World!` in the embedded world seems to be turning on an LED, so that is our first challenge!

## Introduction

There are some [excellent][1] [resources][2] [available][3] with guides on how to turn on the ACT LED for the Raspberry Pi 2, however unfortunately these methods do not work on the Raspberry Pi 3.

Previously, the LEDs were wired directly to GPIO pins and you were able to prod a specific pin to make the LED turn on. Due to space constraints relating to Bluetooth and Wifi, the LEDs were [moved off the main GPIO][4] and onto an expanded—or virtual—GPIO.

The virtual GPIO is controlled by the VideoCore GPU, so we will attempt to turn on the ACT LED by asking the GPU to help us.


## Setup

First things first, you will need the [arm-none-eabi toolchain][5] in order to assemble and generate the final image file.

After that, you'll need the [Raspberry Pi boot files][6]. You only need `bootloader.bin` and `start.elf`. We will generate `kernel.img` ourselves and put all 3 files on a bootable SD card.[^1]

When the Raspberry Pi boots, the process looks roughly like this:

1. The Raspberry Pi powers on and at this point the ARM CPU is **off** and the GPU is **on**.
2. The GPU runs the first bootloader, which is held in ROM on the SoC (System on Chip). This is similar to the BIOS in conventional PCs.
3. This bootloader reads the SD card and loads the second bootloader from `bootloader.bin`.
4. The second bootloader then reads the GPU firmware, also from the SD card, named `start.elf`.
5. Finally, and most importantly for us, `start.elf` reads `kernel.img` and allows the ARM CPU to execute it.

`kernel.img` doesn't have to be an actual kernel (and it won't be for a while), this is just the naming convention.

## First Steps

_The source code (and the final image file) for this post can be found on [GitHub][7], so feel free to grab those and try them out on your own RPi 3._

Firstly, lets create the standard entry point for an assembly file:

```Assembly
.global _start
.section .text
_start:
```

The `.global` identifier tells the assembler to make the symbol `_start` available to the outside world. `.section .text` creates a new section named `.text` which will be used later to define where in our binary we will put our code. Lastly, we make the symbol `_start` take the address of the next line.

## Mailboxes

So, in order to write the next line, we need to know a few things. As I mentioned earlier, we need to communicate with the GPU in order to ask it to turn on the ACT LED. The way we do this is through [mailboxes][8]. Quoting the documentation:

> Mailboxes facilitate communication between the ARM and the VideoCore.

Basically, there are two mailboxes and each is a stack in memory than can be read from (popped) and written to (pushed) by the ARM or the GPU. From the ARM we are able to leave a message in the GPU's mailbox, who will then read it and in turn perform some function[^2].

Now, it's not documented anywhere, but after a [bit of research][9] I found that we are able to control the ACT LED using the [property tags][10] channel. A [bit more digging][11] also lead me the base address of the mailboxes, so now we have everything we need to construct and send a message!

## Preparing to Send a Message

```Assembly
_start:
  mailbox .req r0
  ldr mailbox, =0x3f00b880
```

The first line after `_start:` sets up an alias, allowing us to refer to the register `r0` by the much more useful identifier, `mailbox`. Then we load that register with the base address of the mailboxes.

If we [check the documentation][12], we find that to write to a mailbox we have to:

1. Read the status register until the full flag is not set.
2. Write the data (shifted into the upper 28 bits) combined with the channel (in the lower four bits) to the write register.

So, to continue:

```Assembly
  wait1$:
    status .req r1
```

First of all, we set a label, `wait1$`[^3], as the beginning of our loop, so we know where to come back to. Secondly, we again create an alias `status` for the register `r1`.

We can see that the [status register][13] for mailbox 0 (the _read_ mailbox) is at an offset of `0x18` from the base address.

```Assembly
    ldr status, [mailbox, #0x18]
```

So here we load the address of the status register into `status` (or `r1`).

We need to wait until the status register does _not_ have the full flag set, therefore we need to carry out that check next:

```Assembly
    tst status, #0x80000000
```

The `tst` instruction performs an `AND` between the first operand and the second. In our case, it performs `status AND 0x80000000` and sets the _condition flag_ based on the result. If `status` contains `0x80000000`, the `AND` result will be non-zero which will cause the condition flag to be `ne (not equal)`.

```Assembly
    .unreq status
    bne wait1$
```

If the condition flag is `ne`, `bne wait1$` (**b**ranch **n**ot **e**qual) will branch and jump back to the beginning of the loop. It will continue like this until the status register doesn't have the full flag set. Also, before we exit the loop, we remove the alias `status`.

## Constructing a Message

Let's remind ourselves of the process to send a message:

1. Read the status register until the full flag is not set.
2. Write the data (shifted into the upper 28 bits) combined with the channel (in the lower four bits) to the write register.

The message we leave for the GPU is 32-bits. The first 28-bits are the address of the data we want the GPU to receive and the last 4-bits are the channel we want to send the data to.

If we combine the information we get from the [property tag interface documentation][14] with the [undocumented information][9] I mentioned earlier, we can construct a message to send to the GPU.

Let's create a new section for our message, above the `.text` section:

```Assembly
.section .data
.align 4
PropertyInfo:
```

The `.align 4` instruction tells the assembler to create an address for the following label where the last 4 bits are 0 (this is also called 16-byte aligned). This is necessary so we can use this address in the message we leave in the mailbox.

Then we can define our message data:

```Assembly
  .int PropertyInfoEnd - PropertyInfo
  .int 0

```

First is the message header, which contains the entire size of the data (which we calculate by subtracting the address of the closing label from the address of the leading label) and the request code (which is 0).

```Assembly
  .int 0x00038041
  .int 8
  .int 0
```

Next comes the tag header. This contains the tag ID, the size of the tag data and the request/response size. I believe the final value is written to by the GPU if it needs to respond with data.

```Assembly
  .int 130
  .int 1
```

After the header comes the actual tag data. As we see from the [undocumented source][9], we first include the pin number followed by the pin state.

```Assembly
  .int 0
PropertyInfoEnd:
```

Finally comes the end tag, letting the GPU the message is over, and the closing label (so we can calculate the message size).

## Sending a Message

After all that preparation, we are ready to actually deposit the message into the GPU's mailbox.

```Assembly
  message .req r1
  ldr message, =PropertyInfo
```

Similar to before, we alias `message` to the register `r1` and load it with the address of our message data.

```Assembly
  add message, #8
  str message, [mailbox, #0x20]
  .unreq message
```

Next we add the channel (`8`) into the last 4 bits of the message, as specified by the documentation, and actually put the message into mailbox 1's write register, which is at offset `0x20` from the base address. As we did previously, we remove the alias `message`.

```Assembly
  wait2$:
    b wait2$
```

To finish the file, we give the CPU something to do ad infinitum. If we don't the Raspberry Pi will 'crash', which isn't really a problem for us here as there is nothing else to do!

## Linking

Before we can produce an image, there is one more file we need to write. Don't worry, it's very short.

```
SECTIONS {
  .text 0x8000 : {
    *(.text)
  }

  .data : {
    *(.data)
  }
}
```

This file (`kernel.ld`) lets the linker know where to put our code and data in memory. It seems to be usual for the code to start at address `0x8000`, allowing room for ATAGs and the stack etc, so we put anything in a `.text` section there, followed by anything in a `.data` section.

## Building

I have included a `Makefile` with the source code, but I will quickly run through the necessary steps to produce the final `kernel.img` file.

Firstly, we assemble the source code and produce an object file:

```Bash
arm-none-eabi-as src/kernel.s -o build/kernel.o
```

Secondly, we link those object files with our script and produce an ELF file:

```Bash
arm-none-eabi-ld build/kernel.o -o build/kernel.elf -T kernel.ld
```

Actually, we don't need to specifics that the ELF file contains (like the header) as they are actually instructions to the kernel...and we don't have one!

So finally, we strip the ELF header and generate a binary image:

```Bash
arm-none-eabi-objcopy build/kernel.elf -O binary kernel.img 
```

## So Close

Now we finally have the image file we so worked so hard to get, we can test it out on our Raspberry Pi! Simple copy `kernel.img`, along with `bootloader.bin` and `start.elf`, to a bootable SD and pop it in your Raspberry Pi 3.

If luck is on your side, hopefully the green ACT LED will shine brightly, letting you know you've achieved the impossible.

If darkness greats you, try and try again (or download my [source code][7] and see if that works).

[1]: http://www.cl.cam.ac.uk/projects/raspberrypi/tutorials/os/ok01.html
[2]: http://www.valvers.com/open-software/raspberry-pi/step01-bare-metal-programming-in-cpt1/
[3]: http://blog.thiago.me/raspberry-pi-bare-metal-programming-with-rust/
[4]: https://github.com/raspberrypi/linux/issues/1332#issuecomment-194353863
[5]: https://launchpad.net/gcc-arm-embedded/+download
[6]: https://github.com/raspberrypi/firmware/tree/master/boot
[7]: https://github.com/adamransom/barely_os/releases/tag/bare-metal-1
[8]: https://github.com/raspberrypi/firmware/wiki/Mailboxes
[9]: https://www.raspberrypi.org/forums/viewtopic.php?f=43&t=109137&start=100#p989907
[10]: https://github.com/raspberrypi/firmware/wiki/Mailbox-property-interface
[11]: https://github.com/RobertCNelson/u-boot/blob/8cbb389bb3da80cbf8911f8386cbff92c6a78afe/arch/arm/mach-bcm283x/include/mach/mbox.h#L42
[12]: https://github.com/raspberrypi/firmware/wiki/Accessing-mailboxes
[13]: https://github.com/raspberrypi/firmware/wiki/Mailboxes#mailbox-registers
[14]: https://github.com/raspberrypi/firmware/wiki/Mailbox-property-interface

---

[^1]: `bootloader.bin` and `start.elf` are both closed-source and provided pre-compiled by Broadcom.
[^2]: The GPU can also leave messages in the ARM's mailbox, but we won't be using that feature this time.
[^3]: Appending a `$` to a label is a common convention letting readers know the label is only relevant for the current scope and not important to the program as a whole.