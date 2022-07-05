---
title: 'Building a CLI Tool Aggregator with C#: Beyond Hello World'
date: 2022-07-10T12:06:03+01:00
draft: true
description: "A continuation of the 'commander' series. We're going to build the aggregator by leveraging the fast-cli npm package."
series: ['Commander']
tags: [Dotnet, Node]
---

{{< toc >}}

### System.CommandLine beyond printing "Hello world" {#more-than-hello-world}

Our starting point is getting **`commander`** to run a speed test command and because we need **`commander`** to work with multiple commands we'll be moving away from using the **`SetHandler`** function of our rootCommand object to invoke a command. We'll use individual command objects instead and invoke the commands by calling the **`CommandHandler.Create()`** method on them.

To use the **`CommandHandler`** class, we need to install the **`System.CommandLine.NamingConventionBinder`** package to our project.

```shell
dotnet add package System.CommandLine.NamingConventionBinder --prerelease
```

Our **`Program.cs file`** class needs to be modified for **`cmdr speed`** to print out the results of a speed test to the terminal.

```Program.cs
using System.CommandLine;
using System.CommandLine.NamingConventionBinder;
using System.Diagnostics;


var cmdrRootCommand = new RootCommand();

cmdrRootCommand.Description = "CLI commands aggregator app.";

var speedCommand = new Command("speed", "runs a speed test")
{
    Handler = CommandHandler.Create(() =>
    {
        CommandRunner($"(npm list --global fast-cli || npm install --global fast-cli) && fast --upload --json");
    })
};

cmdrRootCommand.AddCommand(speedCommand);

// Parse the incoming argument and invoke the handler
return cmdrRootCommand.Invoke(args);

static void CommandRunner(string command)
{
    var runProcess = new ProcessStartInfo
    {
        FileName = "pwsh.exe",
        RedirectStandardInput = true,
    };

    Console.WriteLine($"PowerShell process started.");

    var powerShellProcess = Process.Start(runProcess);
    powerShellProcess?.StandardInput.WriteLine(command);
    powerShellProcess?.WaitForExitAsync(default);
    powerShellProcess?.Close();
}
```

The following new things are happening:

1. We introduce **`speedCommand`**, a command object that'll run a speed test when invoked with **`cmdr speed`**.
2. We create a **`Handler`** that represents the action that will be performed when the command is invoked.
3. We use a static method called **`CommandRunner`** to check if the **`fast-cli`** Npm package exists (**`npm list --global fast-cli`**), if it doesn't we install it and then run the speed test command. This is followed by an **`exit`** command to exit the PowerShell process.
4. **`CommandRunner`** uses the **`System.Diagnostics.ProcessStartInfo`** class to start a PowerShell process that'll run our commands.
5. After the handler is created, we add the **`speedCommand`** subcommand to the **`cmdrRootCommand`**.
6. Finally, we invoke **`cmdrRootCommand`**.

> Note: **`System.Diagnostics.Process`** class provides access to local and remote processes and enables you to start and stop local system processes.

Update **`cmdr`** using **`dotnet tool update --global --add-source ./bin/Debug --version 1.0.0 Commander`** and run **`cmdr speed`**

Result:

```shell
PowerShell process started.
PowerShell 7.2.4
Copyright (c) Microsoft Corporation.
https://aka.ms/powershell
Type 'help' to get help.
PS C:\Users\mercymarkus> (npm list --global fast-cli || npm install --global fast-cli) && fast --upload --json
C:\Users\mercymarkus\AppData\Roaming\npm
`--fast-cli@3.2.0
{
"downloadSpeed": 22,
"uploadSpeed": 2.9,
"downloaded": 80,
"uploaded": 10,
"latency": 135,
"bufferBloat": 580,
"userLocation": "Kaduna, NG",
"userIp": "192.180.90.142"
}
```

 <!-- In this case cmd (use powershell core? so it can be used on Mac & linux too. Test installing as a nugetPackage on MacOS without getting visual studio)) -->

Accept command line arguments
Options, extensions, rootcommand, commands, commandHandler, aliases

### JSON

For handling our output as a JSON object, we'll be using a nifty library called **jq**. You can download it [here](https://stedolan.github.io/jq/download/). I downloaded the jq 1.6 executable for windows.

> [jq](https://stedolan.github.io/jq/) is like sed for JSON data - you can use it to slice, filter, map and transform structured data with the same ease that sed, awk, grep and friends let you play with text.

Remember the previous output from running the **`cmdr speed`** command in the last section? We'll be using the **jq** library to filter for only the fields we're interested in (**downloadSpeed, uploadSpeed and latency**) as well as adding extra fields we'd like to collect (**dateTime and connectionType**).

**`speedCommand`** now looks like this:

```csharp
var saveResultOption = new Option<bool>(new[] { "--save-result", "-s" }, getDefaultValue: () => false, "Should speed test result be saved?");

var speedCommand = new Command("speed", "runs a speed test")
{
    Handler = CommandHandler.Create<bool>((saveResult) =>
    {
        var jqFilterCmd = $"fast --upload --json | " +
        $"jq --arg dateTime '{DateTime.Now:yyyy-MM-ddTHH:mm:ss}' " +
        $"'[. | {{downloadSpeed: .downloadSpeed, uploadSpeed: .uploadSpeed, latency:.latency, datetime: $dateTime}}]' | " +
        $"Out-File speed-test-history.json";

        if (saveResult)
        {
            CommandRunner($"(npm list --global fast-cli || npm install --global fast-cli) && {jqFilterCmd}");
        }
        else
        {
            CommandRunner($"(npm list --global fast-cli || npm install --global fast-cli) && fast --upload --json");
        }
    }),
};

cmdrRootCommand.AddCommand(speedCommand);
speedCommand.AddOption(saveResultOption);
```

The additional things happening are:

1. We've added an option called **`saveResultOption`**. It's a Boolean command line option we're using to toggle between 2 states; printing the output of the speed test command to the terminal or saving it to a json file. We'd like to do either (or both?).
2. The **`jqFilterCmd`** variable uses jq to filter the JSON object, includes the extra fields and then outputs this to a file we're calling **`speed-test-history.json`**.
3. Next, we're toggling between the 2 states we mentioned in the first point.
4. Lastly, we're adding **`saveResultOption`** as an option to the **`speedCommand`**.

Note: **`speed-test-history.json`** doesn't need to be hard coded. It can be passed as an option too.

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

### Check Points

- [Checkpoint 1](https://github.com/MercyMarkus/commander/tree/check-point-1)
