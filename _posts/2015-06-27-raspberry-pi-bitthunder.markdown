---
layout: post
title: "Running bitthunder on the raspberry pi"
date: Sat Jun 27 17:00:09 AEST 2015
---

As a part of skyscreen, I've been looking at running a real time operating system on the raspberry pi.

Now, I'm not a developer, so a lot of little tricks were new to me - and I'd guess they're new to you too, so I'll try to document as many of the tricks I used, and hurdles I overcame, to get it to work.

# Bitthunder

I wanted to use [bitthunder](https://github.com/jameswalmsley/bitthunder) on the pi. There were a bunch of reasons:

- Nice docs. Never underestimate the importance of documentation
- From all his posts, james walmsley, the author, seems like a patient and responsive person! Which will be great when I hit problems later on!
- A firm base, from RTOS
- And I'm pretty sure it has network controllers for the pi.

# Building an image

You can clone bitthunder from the repo:

    git clone https://github.com/jameswalmsley/bitthunder.git

You'll need to grab kconfig; all this is documented in bitthunder, so I won't cover it.

*You should definitely donwload the accompanying build tools, from the [btdk repo](https://github.com/bitthunder-toolchain/btdk/releases/tag/btdk-0.6.1). I'm using the 0.6.1 release*

## Configuration options:

These instructions work on the master branch, for comit hash: 0f61a3da2357a78e76f2606a38b028dd5cef2051

Run make menuconfig. I started from the instructions on the bitthunder quickstart, although these are out of date. I'd suggest the follow

	Build System:
	-> Change toolchain-prefix to location of your compiler:
	   e.g.
			/opt/btdk/bin/arm-eabi-bt-

	System Architecture:
	-> Select ARM
	-> Select BCM2835 Chip variant
	-> Select the raspberry pi model B


Then just run make! And let's hope it all worked.

## Adding your own code
*You do need to do this. The default main.c will not do anything interesting. You can easily make it flash an LED, so you don't need a UART*

You'll want to edit: `os/src/bt_main.c`. You can ignore most of it, but down the very bottom, in the main() function, you should add an LED flasher! LED flashers are awesome! Mine looks like this:

{% highlight c %}
#define LED_GPIO_ID 16                                // This is for the pi model B
BT_GpioSetDirection(LED_GPIO_ID, BT_GPIO_DIR_OUTPUT); // Set the pin to be output
int v = 0;

while(1) {
	//BT_kPrint("Welcome to BitThunder");         // I don't care about UART
	BT_ThreadSleep(250);                          // Wait a quarter second
	BT_GpioSet(LED_GPIO_ID, v);                   // Set the output high or low
	v = !v;                                       // and flip the output variable
}
{% endhighlight %}

Run make again! I hope it worked! This creates a few interesting files. The one we want is vmthunder.img

# Uploading to the pi
The pi has an interesting boot system. Here's how to upload your file.

1. Format an SD card with fat32 
2. Download the following files from [the raspberry pi repo](https://github.com/raspberrypi/firmware/tree/master/boot)
  - `start.elf`
  - `bootcode.bin`
3. Copy `start.elf`, `bootcode.bin` to the SD card
4. Copy the created `vmthunder.img` to the SD card *renaming it to `kernel.img`*
5. That's it! Put it in the pi and boot it!

# Results
With any luck, you'll see the LED blinking. And now you know where the code goes, in `os/src/bt_main.c`, so get hacking.

# Troubleshooting

If you had trouble, I've added my own kernel images, that I've confirmed working on my pi. Just decompress onto your SD card.

- [For a raspberry pi model b]({{ site.url }}/assets/pi-b-image.tar.gz)

