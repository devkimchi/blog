---
title: "Turning Raspberry PI into Remote Controller"
slug: turning-raspberry-pi-into-remote-controller
description: "This post shows how to enable LIRC module on a Raspberry PI so that it turns into a remote controller for all home appliances."
date: "2020-08-12"
author: Justin-Yoo
tags:
- raspberry-pi
- remote-controller
- lirc
- air-conditioner
cover: https://sa0blogs.blob.core.windows.net/devkimchi/2020/08/turning-raspberry-pi-into-remote-controller-00.png
fullscreen: true
---

A few months ago, I had a chance to do live streaming with [Cheese (@seojeeee)][twt seojeee] about this topic &ndash; [Part 1][yt lc part1] and [Part 2][yt lc part2] in Korean. I know it's warm and humid in the summer season in Korea. Therefore, I implemented this feature for my air-conditioning system at home, as well as other home appliances that work with remote controllers. However, as I have very little knowledge of Raspberry PI and other hardware, it was really challenging. This is the post to note to future self and others who might be interested in this topic.

* *Turning Raspberry PI into Remote Controller*
* Turning on/off Home Appliances Using Raspberry PI and Power Platform

> The sample codes used in this post can be found at this [GitHub repository][gh sample].


## Check Hardware and Software Specs ##

It might be only me, but I found that [Raspberry PI][rpi] and its extension modules like IR sensor are very version sensitive. I Google'd many relevant articles on the Internet, but they are mostly outdated &ndash; not valid information any longer. Of course, this post will also have a high chance to be obsolete. Therefore, to avoid visitors from being disappointed in the future, it would be a great idea that I specify which hardware and software spec I used.

* Hardware
  * [Raspberry PI 2 Model B v1.1][rpi 2b]
  * [IR Remote Shield v1.0][rpi ir remote shield]
* Software
  * [Raspberry PI OS LITE][rpi os lite]: Debian 10 (Buster) base, released on 2020-05-27
  * [LIRC (Linux Infrared Remote Control)][lirc]: v0.10.1, released on 2017-09-10


## LIRC Module Installation ##

The very first step is to install LIRC after setting up the Raspberry PI OS. Enter the command below to install LIRC.

https://gist.github.com/justinyoo/34e3470ced2641221c69ebeab6e034f7?file=01-install-lirc.sh

Your Raspberry PI OS has now been up-to-date and got the LIRC module installed.

![][image-01]


## LIRC Module Configuration ##

Let's configure the LIRC module to send and receive the IR signal.


### Bootloader Configuration ###

By updating the bootloader file, when Raspberry PI starts the LIRC module starts at the same time. Open the bootloader file:

https://gist.github.com/justinyoo/34e3470ced2641221c69ebeab6e034f7?file=02-modify-boot-config.sh

Uncomment the following lines and correct the pin number. The default values before being uncommented were 17 for `gpio-ir` and 18 for `gpio-ir-tx`. But it should be swapped (line #5-6).

> Of course, it might be working without swapping the pin numbers. But it wasn't my case at all. I had to change them to each other.

https://gist.github.com/justinyoo/34e3470ced2641221c69ebeab6e034f7?file=03-boot-config.sh&highlights=5-6


### LIRC Module Hardware Configuration ###

Let's configure the LIRC module hardware. Open the file below:

https://gist.github.com/justinyoo/34e3470ced2641221c69ebeab6e034f7?file=04-modify-lirc-hardware.sh

Then enter the following:

https://gist.github.com/justinyoo/34e3470ced2641221c69ebeab6e034f7?file=05-lirc-hardware.sh


### LIRC Module Options Configuration ###

Update the LIRC module options. Open the file with the command:

https://gist.github.com/justinyoo/34e3470ced2641221c69ebeab6e034f7?file=06-modify-lirc-lircoptions.sh

Change both `driver` and `device` values (line #3-4).

https://gist.github.com/justinyoo/34e3470ced2641221c69ebeab6e034f7?file=07-lirc-lircoptions.sh&highlights=3-4

Once you've completed by now, reboot Raspberry PI to recognise the updated bootloader.

https://gist.github.com/justinyoo/34e3470ced2641221c69ebeab6e034f7?file=08-reboot.sh

Send this command to check the LIRC module working or not.

https://gist.github.com/justinyoo/34e3470ced2641221c69ebeab6e034f7?file=09-lircd-status.sh

![][image-02]

It's now working!


## Remote Controller Registration ##

This is the most important part of all. I have to register the remote controller I'm going to use.


### Use Remote Controller Database for Registration ###

The easiest and promising way to register the remote controller is to visit [Remote Controller Database website][lirc remote], search the remote controller and download it. For example, as long as you know the remote controller model name, it can be found on the database. I just searched up any LG air-conditioner.

![][image-03]

If you can find your remote controller details, download it and copy it to the designated location.

https://gist.github.com/justinyoo/34e3470ced2641221c69ebeab6e034f7?file=10-copy-lircd-conf.sh


### Manual Registration for Remote Controller ###

Not every remote controller has been registered to this database. If you can't find yours, you should create it by yourself. Winia makes my air-conditioning system, but it doesn't exist on the database. I've got a local branded TV which doesn't exist on the database either. I had to create them. To create the configuration file, Raspberry PI should be double-checked whether the IR sensor on it captures the remote controller signal or not. First of all, stop the LIRC service.

https://gist.github.com/justinyoo/34e3470ced2641221c69ebeab6e034f7?file=11-lircd-stop.sh

![][image-04]

Run the following command to wait for the incoming IR signals.

https://gist.github.com/justinyoo/34e3470ced2641221c69ebeab6e034f7?file=12-mode2.sh

If you're unlucky, you'll get the following error message. Yeah, it's me.

![][image-05]

It's because both the IR sender and receiver are active. At this time, we only need the receiver, which is for capturing the IR signals. Disable the sender part. Open the bootloader.

https://gist.github.com/justinyoo/34e3470ced2641221c69ebeab6e034f7?file=13-modify-boot-config.sh

We used to have both `gpio-ir` and `gpio-ir-tx` activated. As we don't need the sender part, for now, update the file like below (line #5-6).

https://gist.github.com/justinyoo/34e3470ced2641221c69ebeab6e034f7?file=14-boot-config.sh&highlights=5-6

Once completed, reboot Raspberry PI using the command, `sudo reboot`. Once it's restarted, stop the LIRC server.

https://gist.github.com/justinyoo/34e3470ced2641221c69ebeab6e034f7?file=11-lircd-stop.sh

![][image-04]


Then, run the following command so that you can confirm it works.

https://gist.github.com/justinyoo/34e3470ced2641221c69ebeab6e034f7?file=15-mode2.sh

![][image-06]

Now it's waiting for your IR signal input. Locate your remote controller close to Raspberry PI and punch some buttons. You'll find out the remote controller buttons are captured.

https://youtu.be/WrTlpCHl1ZA

<br/>We confirm the incoming signals are properly captured. It's time to generate the remote controller file. Enter the following command:<br/>

https://gist.github.com/justinyoo/34e3470ced2641221c69ebeab6e034f7?file=16-irrecord.sh

Once running the command above, it gives you instructions to follow. Record your buttons by following the instruction. By the way, the recording application sometimes doesn't work as expected. Yes, I was in the case. I had to use a different approach. Instead of using `irrecord`, I had to use `mode2` to capture the button signals. Run the following command:

https://gist.github.com/justinyoo/34e3470ced2641221c69ebeab6e034f7?file=17-mode2.sh

Record the button signals. When you open the `<remote_controller_name>.lircd.conf` file, you'll be able to see some number blocks.

![][image-07]

The number blocks in the red rectangle are the set of the controller button. As the last value of the box is an outlier, delete it. And remove all others except the number blocks. Let's add the followings around the number blocks.

* Add `begin remote ... begin raw_codes` before the fist number block. They are from the random file on the database. I don't know what the exact value might look like. I just copied and pasted them to the file (line #1-13).
* Give each number block a name like `name SWITCH_ON`. Each button has a different value represented by the number block. Therefore give those names extensively (line #15, 22).
* Add the lines at the end of the final number block (line #29-30).

https://gist.github.com/justinyoo/34e3470ced2641221c69ebeab6e034f7?file=18-lircd-conf.txt&highlights=1-13,15,22,29-30

After this update, copy this file to the LIRC directory.

https://gist.github.com/justinyoo/34e3470ced2641221c69ebeab6e034f7?file=19-copy-lircd-conf.sh

When you go to the directory, you'll be able to find those files. I've got both files registered for an air-conditioner and TV.

![][image-08]

As we don't need `devinput.lircd.conf` any longer, rename it.

https://gist.github.com/justinyoo/34e3470ced2641221c69ebeab6e034f7?file=20-rename-devinput-lircd-conf.sh

Remote controllers have been registered. Open the bootloader for the update.

https://gist.github.com/justinyoo/34e3470ced2641221c69ebeab6e034f7?file=21-modify-boot-config.sh

Update the IR sender part (line #5-6).

https://gist.github.com/justinyoo/34e3470ced2641221c69ebeab6e034f7?file=22-boot-config.sh&highlights=5-6

Run `sudo reboot` to reboot Raspberry PI. Check whether the LIRC module is working or not.

https://gist.github.com/justinyoo/34e3470ced2641221c69ebeab6e034f7?file=23-lircd-status.sh

We can confirm that the LIRC module read both air-conditioner and TV configurations and ready for use!

![][image-09]

Theoretically, we can register as many remote controllers as we like!


## Controlling Air-Conditioner and TV on Raspberry PI ##

Let's check which commands the remote controller on Raspberry PI have. Enter the following command to see the list of names that I can execute.

https://gist.github.com/justinyoo/34e3470ced2641221c69ebeab6e034f7?file=24-irsend-list.sh

I've registered both air-conditioner (`winia`) and TV (`tv`). The following screenshot is the result I've got. Each remote controller has more buttons, but I only need two buttons &ndash; `on` and `off`. Therefore, I only registered those two buttons.

![][image-10]

OK. Let's run the command.

https://gist.github.com/justinyoo/34e3470ced2641221c69ebeab6e034f7?file=25-irsend-sendonce.sh

Although the terminal shows nothing, I actually turn on and off the TV.

![][image-11]

Can you see the TV being turned on and off?

https://youtu.be/QoUmSVAxBCs

<br/>Unfortunately, I can't make my air-conditioner be working. I might have incorrectly captured the IR signal for the air-conditioner. But TV works, instead! To me, the air-conditioner was the top priority, though. I should spend more time to get it working. If I have a more sophisticated device to capture the IR signal, it would be better.

---

So far, we have walked through how [Raspberry PI][rpi] turns into a remote controller that turns on and off home appliances. In the next post, I'll build a [Power App][pw apps] and [Power Automate][pw automate] that talks to [Azure Functions][az func] app to access to the remote controller (Raspberry PI) outside the home network.


[image-01]: https://sa0blogs.blob.core.windows.net/devkimchi/2020/08/turning-raspberry-pi-into-remote-controller-01.png
[image-02]: https://sa0blogs.blob.core.windows.net/devkimchi/2020/08/turning-raspberry-pi-into-remote-controller-02.png
[image-03]: https://sa0blogs.blob.core.windows.net/devkimchi/2020/08/turning-raspberry-pi-into-remote-controller-03.png
[image-04]: https://sa0blogs.blob.core.windows.net/devkimchi/2020/08/turning-raspberry-pi-into-remote-controller-04.png
[image-05]: https://sa0blogs.blob.core.windows.net/devkimchi/2020/08/turning-raspberry-pi-into-remote-controller-05.png
[image-06]: https://sa0blogs.blob.core.windows.net/devkimchi/2020/08/turning-raspberry-pi-into-remote-controller-06.png
[image-07]: https://sa0blogs.blob.core.windows.net/devkimchi/2020/08/turning-raspberry-pi-into-remote-controller-07.png
[image-08]: https://sa0blogs.blob.core.windows.net/devkimchi/2020/08/turning-raspberry-pi-into-remote-controller-08.png
[image-09]: https://sa0blogs.blob.core.windows.net/devkimchi/2020/08/turning-raspberry-pi-into-remote-controller-09.png
[image-10]: https://sa0blogs.blob.core.windows.net/devkimchi/2020/08/turning-raspberry-pi-into-remote-controller-10.png
[image-11]: https://sa0blogs.blob.core.windows.net/devkimchi/2020/08/turning-raspberry-pi-into-remote-controller-11.png

[twt seojeee]: https://twitter.com/seojeee

[yt lc part1]: https://youtu.be/q8TuRTw0OsQ
[yt lc part2]: https://youtu.be/2iaSu9kOQGE

[gh sample]: https://github.com/devkimchi/Raspberry-PI-Remote-Controller-Sample

[rpi]: https://www.raspberrypi.org/
[rpi 2b]: https://www.raspberrypi.org/products/raspberry-pi-2-model-b/
[rpi ir remote shield]: https://www.amazon.com/IR-Remote-Control-Transceiver-Raspberry/dp/B0713SK7RJ
[rpi os lite]: https://www.raspberrypi.org/downloads/raspberry-pi-os/

[lirc]: https://www.lirc.org/
[lirc remote]: http://lirc-remotes.sourceforge.net/remotes-table.html

[az func]: https://docs.microsoft.com/azure/azure-functions/functions-overview?WT.mc_id=devkimchicom-blog-juyoo

[pw automate]: https://flow.microsoft.com/?WT.mc_id=devkimchicom-blog-juyoo
[pw apps]: https://powerapps.microsoft.com/?WT.mc_id=devkimchicom-blog-juyoo
