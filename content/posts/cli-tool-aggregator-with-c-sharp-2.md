---
title: 'Building a CLI Tool Aggregator with C#: Beyond Hello World'
date: 2022-04-28T12:13:30+05:30
draft: false
description: "What began as me writing an article about a new thing I'd learnt turned into me trying to build something that at least 1 person would find useful. I am the 'one' person. 😄"
series: ['Commander']
tags: [Dotnet, Node]
---

Todo: Use different branches to checkpoint different states of the code.

{{< toc >}}

### Using System.CommandLine to make our tool do more than print Hello world

Accept command line arguments
Options, extensions, rootcommand, commands, commandHandler, aliases

### Run node processes in our app

Use ProcessStart Info to install and run the speed-test command. i.e `commander speed` should run `speed-test --json` in the background.

### Introduce jq and use jq to build the json object

### NetworkInterface.GetAllNetworkInterfaces() to check if network if wired or wireless

### Add other commands to commander

### Future Work

Save to the cloud and plot historic graph of network speeds.
Add location field to the json blob. Where was I?

There are already a bunch of CLI tools that do similar things:

- speedtest.net even saves the data for you if your create an account. I'll most likely use this more because It's web based and works directly in the browser too. It also has a CLI tool as well, but you can't save from there to your account. Same for the mobile app.
- What started as me checking if I could call the speedtest.com or fast.com website from Postman took me down a rabbit hole. I found lots of Speed test CLI tools.
- Found a cool tool that logs the speed every x interval. Might be nice to add (npm install --global internet-speed-continuous-monitor; https://www.npmjs.com/package/internet-speed-continuous-monitor)
- Add ISP (https://www.npmjs.com/package/netspeedchecker)
