---
title: 'Building a CLI Tool Aggregator with C#: Beyond Hello World'
date: 2022-07-19T12:06:03+01:00
draft: false
description: "A continuation of the 'commander' series. We're going to leverage the fast-cli npm package."
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
        CommandRunner($"(npm list --global fast-cli || npm install --global fast-cli) && fast --json");
    })
};

cmdrRootCommand.AddCommand(speedCommand);

// Parse the incoming argument and invoke the handler
return cmdrRootCommand.Invoke(args);

static void CommandRunner(string command)
{
    var runProcess = new ProcessStartInfo("pwsh.exe", $"-Command {command}")
    {
        RedirectStandardInput = true,
        RedirectStandardOutput = true,
        RedirectStandardError = true,
        WorkingDirectory = Environment.GetFolderPath(Environment.SpecialFolder.UserProfile),
    };

    Console.WriteLine($"PowerShell process started.");

    var process = Process.Start(runProcess);
    Console.WriteLine(process?.StandardOutput.ReadToEnd());
    process?.WaitForExit();

    if (process?.ExitCode != 0)
    {
        throw new Exception($"cmdr encountered an issue: {process?.StandardError.ReadToEnd()}");
    }

    process?.Close();
}
```

The following new things are happening:

1. We introduce **`speedCommand`**, a command object that'll run a speed test when invoked with **`cmdr speed`**.
2. We create a **`Handler`** that represents the action that will be performed when the command is invoked. In this case, calling **`CommandRunner`**.
3. **`CommandRunner`** checks if the **`fast-cli`** npm package exists (**`npm list --global fast-cli`**), if it doesn't we install it and then run the speed test command.
4. **`CommandRunner`** uses the **`System.Diagnostics.ProcessStartInfo`** class to start a PowerShell process that'll run our commands.
5. We're setting some defaults for the process that gets started. These are:
   1. **Redirecting the standard input**: Gets or sets a value indicating whether the input for an application is read from the `System.Diagnostics.Process.StandardInput` stream.
   2. **Redirecting the standard output**: Gets or sets a value that indicates whether the textual output of an application is written to the `System.Diagnostics.Process.StandardOutput` stream.
   3. **Redirecting the standard error**: Gets or sets a value that indicates whether the error output of an application is written to the System.Diagnostics.Process.StandardError stream.
   4. **Working Directory**: Gets or sets the working directory for the process to be started. Here we're setting it to the default UserProfile (in my case **`Environment.SpecialFolder.UserProfile`** = **`C:\Users\mercymarkus`**). This is so our speed test results get saved to a known location and we're able to build a database.
6. After the process is started, we print out every output to the console, wait for the process to exit, and then close the process. If it exits with an exitCode that isn't 0, we throw an exception and print it out to the terminal (0 means our code ran without any problems).
7. After the handler is created, we add the **`speedCommand`** subcommand to the **`cmdrRootCommand`**.
8. Finally, we invoke **`cmdrRootCommand`**.

> Note: **`System.Diagnostics.Process`** class provides access to local and remote processes and enables you to start and stop local system processes.

We'll update **`cmdr`** using **`dotnet tool update --global --add-source ./bin/Debug --version 1.0.0 Commander`** and run **`cmdr speed`** to test these changes.

The result is:

```shell
PowerShell process started.
C:\Users\mercymarkus\AppData\Roaming\npm
`-- fast-cli@3.2.0
{
        "downloadSpeed": 5.8,
        "downloaded": 9.3,
        "latency": 129,
        "bufferBloat": 331,
        "userLocation": "Kaduna, NG",
        "userIp": "108.86.39.54"
}
```

### Handle JSON data with jq

For handling our output as a JSON object, we'll be using a nifty library called **jq**. You can download it [here.](https://stedolan.github.io/jq/download/) I downloaded the jq 1.6 executable for windows.

> [jq](https://stedolan.github.io/jq/) is like sed for JSON data - you can use it to slice, filter, map and transform structured data with the same ease that sed, awk, grep and friends let you play with text.

Remember the previous output from running the **`cmdr speed`** command in the last section? We'll be using the **jq** library to filter out the fields we're interested in (**downloadSpeed, uploadSpeed, and latency**) as well as adding extra fields we'd like to collect (**dateTime and connectionType**) and then export this JSON as a CSV we're using to create a database of our speed test results over time.

**`speedCommand`** now looks like this:

```csharp
var saveResultOption = new Option<bool>(new[] { "--save-result", "-s" }, getDefaultValue: () => false, "Should speed test result be saved?");
var fileNameOption = new Option<string>(new[] { "--filename", "-f" }, getDefaultValue: () => "speed-results", "JSON & CSV file name for speed test results.");

var speedCommand = new Command("speed", "runs a speed test")
{
    Handler = CommandHandler.Create<bool, string>((saveResult, fileName) =>
    {
        if (saveResult)
        {
            var formatDateTime = $"{DateTime.Now:yyyy-MM-ddTHH:mm:ss}";
            var constructSpeedTestObject = "{downloadSpeed: .downloadSpeed, uploadSpeed: .uploadSpeed, latency: .latency, " +
            "datetime: $dateTime, connectionType: $connectionType, location: .userLocation}";

            var filterSpeedTestOutput = $"jq --arg dateTime '{formatDateTime}' --arg connectionType '{GetConnectionType()}' '. | {constructSpeedTestObject}'";
            var saveSpeedTestOutput = $"fast --upload --json | {filterSpeedTestOutput} | Tee-Object -FilePath {fileName}.json -Append";
            var createCsvWithJq = "jq -r '(map(keys) | add | unique) as $cols | map(. as $row | $cols | map($row[.])) as $rows | $cols, $rows[] | @csv'";
            var createSpeedTestCsv = $"Get-Content -Path .\\{fileName}.json -Raw | jq -s . | {createCsvWithJq} | Out-File {fileName}.csv";

            CommandRunner($"(npm list --global fast-cli || npm install --global fast-cli) && {saveSpeedTestOutput} && {createSpeedTestCsv}");
        }
        else
        {
            CommandRunner($"(npm list --global fast-cli || npm install --global fast-cli) && fast --json");
        }
    }),
};

cmdrRootCommand.AddCommand(speedCommand);
speedCommand.AddOption(saveResultOption);
speedCommand.AddOption(fileNameOption);
```

The additional things happening are:

1. We've added an option called **`saveResultOption`**. It's a Boolean command line option we're using to toggle between 2 states; printing the output of the speed test command to the terminal or printing and then saving it to a JSON file which gets exported as a CSV.
2. We've also added a **`fileNameOption`**, a string option that customizes the speed test result filenames. We're setting a default value(**`getDefaultValue: () => "speed-results"`**) so we skip adding the **`--filename` or `-f`** flag at execution time. This saves us some keystrokes while still allowing the users of the tool to use a custom value.
3. In the **`if`** statement block, the following happens:
   1. We're creating a filter that includes all the fields we'd like added to our JSON object. This happens in **`var constructSpeedTestObject`**.
   2. We're using jq to filter the JSON object (**`var filterSpeedTestOutput`**) and passing the fields we're creating as arguments (**`dateTime` and `$connectionType`**).
   3. The arguments format the current dateTime value and invoke the **`GetConnectionType()`** method (we'll talk about this in the next section).
   4. We run the speed test command next and pass the output to PowerShell's **`Tee-Object`** function. This prints the result to the terminal and also appends it to a file. The default file name is **`speed-test.json`**.
   5. In the **`createCsvWithJq`** variable, jq is used to create a CSV file by mapping the keys and values of the JSON object as rows and columns.
   6. The **`createSpeedTestCsv`** variable gets the contents of the **`speed-test.json`**, creates the CSV using jq, and then saves the output as **`speed-test.csv`**
   7. Lastly, CommandRunner runs the speed test command and saves the output as a CSV.
4. The else block executes the speed test command (without saving to a JSON/CSV file) if the **`saveResultOption`** is **false**.
5. Lastly, we're adding **`saveResultOption`** and **`fileNameOption`** as options to the **`speedCommand`** command.

### Get network connection type

We're using the **`NetworkInterface.GetAllNetworkInterfaces()`** method to get the network connection type. It returns an object that describes the network interfaces available on our local computer. We can then iterate through the interfaces that are operational and are either wireless or ethernet interfaces.
We have no interest in vEthernet, hence the exclusion.

> vEthernet switches allow network access for virtual machines and other aspects of Hyper-V.

```csharp
static string GetConnectionType()
{
    var connectionType = string.Empty;

    NetworkInterface[] adapters = NetworkInterface.GetAllNetworkInterfaces();

    foreach (NetworkInterface adapter in adapters.Where(a => a.OperationalStatus == OperationalStatus.Up
        && (a.NetworkInterfaceType == NetworkInterfaceType.Wireless80211 || a.NetworkInterfaceType == NetworkInterfaceType.Ethernet)
        && !a.Name.StartsWith("vEthernet")))
    {
        if (adapter.NetworkInterfaceType == NetworkInterfaceType.Wireless80211)
        {
            connectionType = adapter.NetworkInterfaceType.ToString().Substring(0, 8);
        }
        else
        {
            connectionType = adapter.NetworkInterfaceType.ToString();
        }
    }
    return connectionType;
}
```

### Extra: Change commands to LowerCase

This additional change was added to avoid **`CMDR SPEED`** from not executing. I noticed that the root command was case-insensitive but not the subcommands i.e **`CMDR` or `CmDr`** work but not **`CMDR SPEED` or `CmDr Speed`** or other variants.

> [Here's why commands, option names, and aliases are case-sensitive by default.](https://docs.microsoft.com/en-us/dotnet/standard/commandline/syntax#case-sensitivity)

Output of **`cmdr Speed`** before adding and calling the **`ChangeCommandsToLowerCase()`** method:

```shell
Required command was not provided.
Unrecognized command or argument 'Speed'.
Description:
    CLI commands aggregator app.

Usage:
    commander [command] [options]

Options:
    --version       Show version information
    -?, -h, --help  Show help and usage information

Commands:
    speed  runs a speed test
```

The **`ChangeCommandsToLowerCase()`** method:

```csharp
static void ChangeCommandsToLowerCase(string[] args)
{
    for (int i = 0; i < args.Length; i++)
    {
        args[i] = args[i].ToLower();
    }
}
```

This method is called right before invoking **`cmdrRootCommand`**. It takes in the same arguments as the root command, iterates through the list of arguments, and changes them to lower case.

While building this, I discovered that there were a bunch of CLI tools that do similar things. Some honorable mentions are:

1. **speedtest.net**: It saves your speed test results if you create an account. It's web-based and works directly in the browser and also has a CLI tool. The downside for me is that I can't save speed test results from the mobile app or CLI tool to my account. It has to be from the browser and that's not ideal because I reach for my terminal more frequently than a browser.
2. **internet speed continuous monitor npm package**: It's an npm package that logs the internet speed every x interval.

Finally, if you got this far and you're wondering where the CLI tool aggregation is happening as the title says, I'll cover this in the next post 🙏🏾. This was getting too long.

I plan to save the speed test results in some online storage location (undecided on which) and update a live graph of the results in real-time as well.

### Check Point

- [Second checkpoint](https://github.com/MercyMarkus/commander/tree/check-point-2)
