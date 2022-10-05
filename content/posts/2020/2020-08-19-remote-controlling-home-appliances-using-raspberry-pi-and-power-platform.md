---
title: "Turning on/off Home Appliances Using Raspberry PI and Power Platform"
slug: remote-controlling-home-appliances-using-raspberry-pi-and-power-platform
description: "This post shows how to turn on and off home appliances from the Internet, via Power Platform."
date: "2020-08-19"
author: Justin-Yoo
tags:
- raspberry-pi
- remote-controller
- power-platform
- azure-functions
cover: /2020/08/remote-controlling-home-appliances-using-raspberry-pi-and-power-platform-00.png
fullscreen: true
---

In my [previous post][post prev], we discussed how [Raspberry PI][rpi] can turn into a remote controller, using the [LIRC][lirc] module. Throughout this post, I'll be literally remote controlling even outside the home network, using [Power Platform][pw platform].

* [Turning Raspberry PI into Remote Controller][post prev]
* ***Turning on/off Home Appliances Using Raspberry PI and Power Platform***

> The sample codes used in this post can be found at this [GitHub repository][gh sample].


## node.js API App Server ##

There is an npm package, called [node-lirc][npm node-lirc]. With this package, we can easily build a node.js application to handle the LIRC command.


### Remote Controller Function ###

Let's write a simple `remote.js` module. Create a function, called `remote()`. It takes the device name and command (line #5-9). What it does is to just run the LIRC command, `irsend SEND_ONCE <device> <command>` (line #6). Then, create a command object of `remoteControl` that contains `onSwitchOn` and `onSwitchOff` functions.

https://gist.github.com/justinyoo/352f186bf49fece4cb3dfb4cfae45cab?file=01-remote.js&highlights=5-9


### API Server Application ###

We've got the remote controller module. This time, let's build an API app that uses the remote controller module. If we use the [express][npm express] package, we can build the API app really quickly. Let's have a look at the code below. it defines both endpoints for turning on and off appliances, `/remote/switchon` and `/remote/switchoff` respectively (line #6, 17). This example uses the `GET` verb, but strictly speaking, the `POST` verb is way closer. Finally, it allocates the port number of 4000 for this application (line #28).

https://gist.github.com/justinyoo/352f186bf49fece4cb3dfb4cfae45cab?file=02-app.js&highlights=6,17,28

Now we've got the API server app ready to run. Double check `package.json` so that the API can be run with the following command (line #4).

https://gist.github.com/justinyoo/352f186bf49fece4cb3dfb4cfae45cab?file=03-package.json&highlights=4

Run the command, `npm run start` to run the API app.

![][image-01]

The API app is ready to receive requests. Open a web browser and send a request for the remote controller.

https://youtu.be/ULZ2KYZejWw

<br/>
We've now got the API app up and running for the remote controller.


## Secure Tunnelling to Home Network &ndash; ngrok ##

The API running on Raspberry PI resides within the home network. It means that, unless we open a secure channel, it can't be accessible from the Internet because my home network uses the private IP address range like `172.30.xxx.xxx`. There are many ways to sort out this issue, but I'm going to use [ngrok][ngrok] to open a secure tunnel that allows only specified services. For more details, refer to their [official documents][ngrok docs]. First of all, [download the application and install it][ngrok download], then run the following command to open a secure tunnel.

https://gist.github.com/justinyoo/352f186bf49fece4cb3dfb4cfae45cab?file=04-run-ngrok.sh

The UI of ngrok looks like the following picture. As we use its free version, we can't use a custom domain but are only allowed the randomly generated URL, every time we run it. The picture says it's now `https://48b6ff595189.ngrok.io`, but the next time, it'll be different.

![][image-02]

With this ngrok URL, we can access to the home network from the Internet. Let's have a look at the video:

https://youtu.be/nT-CXCueGEc


## Controlling Remote Controller with Azure Functions ##

The purpose of the secure tunnelling is to run the remote controller from [Azure Functions][az func] on the Internet. We can quickly build an Azure Functions app with node.js. The following function code uses either query string parameters or request body payload to populate both `device` and `power` values (line #6-7). In addition to that, it gets the ngrok-generated URL from the environment variables (line #9-11). By combining them (line #14), the function app calls the remote controller on Raspberry PI (line #16).

https://gist.github.com/justinyoo/352f186bf49fece4cb3dfb4cfae45cab?file=05-function.js&highlights=6-7,9-11,14,16

Let's run the function app on my local machine and send a request to the remote controller through ngrok.

https://youtu.be/o9Rhwktzvgs

<br/>
Deploy the function app to Azure and run it again. Everything goes alright.<br/><br/>

https://youtu.be/r2lX6_qxXAE


## Controlling Remote Controller through Power Platform ##

[Azure Functions][az func] simply works as a proxy that calls the ngrok-secured remote controller API. It's OK we stop right here, as long as we use a web browser. But let's be fancier, using [Power Platform][pw platform]. [Power Automate][pw automate] handles the workflow of the remote controller by calling the Azure Functions app and [Power Apps][pw apps] defines the behaviours how to "remote control" home appliances.

We can call Azure Functions directly from Power Apps, but the separation of concerns is also applied between [Power Automate][pw automate] and [Power Apps][pw apps] &ndash; control the workflow and control the behaviours. Let's build the workflow first.


## Power Automate Workflow ##

First of all, create a workflow instance at Power Automate. As Power App only triggers it, it sets the Power App as a trigger.

![][image-03]

Once the instance is created, it opens the visual designer screen. Can you see the Power App trigger at the top?

![][image-04]

As this workflow simply calls the Azure Functions app, search up the HTTP action.

![][image-05]

We created the Azure Functions endpoint to accept both `GET` and `POST` verbs. When we test it on the browser, we had to send the request through the `GET` verb. In Power Automate, it's better to use the `POST` verb to send requests to the function app.

Set the function app endpoint URL and header like below. You can see both `HTTP_Body` and `HTTP_Body_1` at the bottom-right side corner. Both represent device and power, respectively, which will be explained later in this post.

![][image-07]

Search up the HTTP response action to return the execution result from the function app endpoint to Power App.

![][image-08]

Put the entire HTTP action response to the response action.

![][image-09]

Click the `Show advanced options` button to define a JSON schema for Power Apps to recognise what the response might look like:

![][image-10]

Click the `Generate from sample` button and paste the response payload taken from the previous example.

![][image-11]

Then, it automatically generates the JSON schema.

![][image-12]

We've now got the [Power Automate][pw automate] workflow that calls the Azure Functions endpoints.


### Power App Remote Controller for All Home Appliances ###

Let's build the "real" remote controller with [Power Apps][pw apps]. Add two buttons and one label controls. Set the name of "ON" to the first button and "OFF" to the second button.

![][image-13]

Connect the [Power Automate][pw automate] workflow of `Home Appliances Remote Controller` with the `ON` button. The `HomeAppliancesRemoteController.Run()` function represents the workflow that takes two parameters of `tv` and `on`. Do you remember both `HTTP_Body` and `HTTP_Body_1` from the Power Automate workflow that takes device and power respectively? These `tv` and `on` are the ones.

![][image-14]

Like the `ON` button, the `OFF` button calls the same workflow with different parameters &ndash; `tv` and `off`.

![][image-15]

The text label shows the result of the call.

![][image-16]

We just use only one appliance, that is TV. But if we add two extra buttons for an air-conditioning system, we can send parameters like `winia` and `on`, or `winia` and `off`.

Save and publish the Power App.

![][image-17]

On our mobile phone, the Power App might look like this video:

https://youtu.be/iI6nfwwzFQ4

<br/>
If I run this app in front of the TV, it looks like the following video:<br/><br/>

https://youtu.be/JXPLTuloSWg

---

So far, we built an API app, opened a secure tunnel, and built an Azure Function app, Power Automate workflow and Power App. Through this Power App, we can now control our home appliances outside the home. As there are plenty of scenarios for Power Platform, I would really like you to run your idea. Are you fascinated? It's your turn now.


[image-01]: /2020/08/remote-controlling-home-appliances-using-raspberry-pi-and-power-platform-01.png
[image-02]: /2020/08/remote-controlling-home-appliances-using-raspberry-pi-and-power-platform-02.png
[image-03]: /2020/08/remote-controlling-home-appliances-using-raspberry-pi-and-power-platform-03.png
[image-04]: /2020/08/remote-controlling-home-appliances-using-raspberry-pi-and-power-platform-04.png
[image-05]: /2020/08/remote-controlling-home-appliances-using-raspberry-pi-and-power-platform-05.png
[image-06]: /2020/08/remote-controlling-home-appliances-using-raspberry-pi-and-power-platform-06.png
[image-07]: /2020/08/remote-controlling-home-appliances-using-raspberry-pi-and-power-platform-07.png
[image-08]: /2020/08/remote-controlling-home-appliances-using-raspberry-pi-and-power-platform-08.png
[image-09]: /2020/08/remote-controlling-home-appliances-using-raspberry-pi-and-power-platform-09.png
[image-10]: /2020/08/remote-controlling-home-appliances-using-raspberry-pi-and-power-platform-10.png
[image-11]: /2020/08/remote-controlling-home-appliances-using-raspberry-pi-and-power-platform-11.png
[image-12]: /2020/08/remote-controlling-home-appliances-using-raspberry-pi-and-power-platform-12.png
[image-13]: /2020/08/remote-controlling-home-appliances-using-raspberry-pi-and-power-platform-13.png
[image-14]: /2020/08/remote-controlling-home-appliances-using-raspberry-pi-and-power-platform-14.png
[image-15]: /2020/08/remote-controlling-home-appliances-using-raspberry-pi-and-power-platform-15.png
[image-16]: /2020/08/remote-controlling-home-appliances-using-raspberry-pi-and-power-platform-16.png
[image-17]: /2020/08/remote-controlling-home-appliances-using-raspberry-pi-and-power-platform-17.png

[post prev]: /2020/08/12/turning-raspberry-pi-into-remote-controller/

[gh sample]: https://github.com/devkimchi/Raspberry-PI-Remote-Controller-Sample

[rpi]: https://www.raspberrypi.org/
[lirc]: https://www.lirc.org/
[nodejs]: https://nodejs.org/
[ngrok]: https://ngrok.com/
[ngrok docs]: https://ngrok.com/docs
[ngrok download]: https://ngrok.com/download

[npm node-lirc]: https://www.npmjs.com/package/node-lirc
[npm express]: https://www.npmjs.com/package/express

[az func]: https://docs.microsoft.com/azure/azure-functions/functions-overview?WT.mc_id=devkimchicom-blog-juyoo

[pw platform]: https://powerplatform.microsoft.com/?WT.mc_id=devkimchicom-blog-juyoo
[pw automate]: https://flow.microsoft.com/?WT.mc_id=devkimchicom-blog-juyoo
[pw apps]: https://powerapps.microsoft.com/?WT.mc_id=devkimchicom-blog-juyoo
