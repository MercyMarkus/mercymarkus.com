---
title: 'Building a CLI Tool Aggregator with C#: Creating a hello world .NET tool'
date: 2022-04-18T12:13:30+05:30
draft: false
description: "What began as me writing an article about a new thing I'd learnt turned into me trying to build something that at least 1 person would find useful. I am the 'one' person. 😄"
series: ['Commander']
tags: [Dotnet]
---

Todo: Use different branches to checkpoint different states of the code.

{{< toc >}}

Coming from a Python background, getting a hang of C# was daunting. The syntax was verbose, had lots filler code and setting up projects was more involved in comparison to Python.

Overtime, I've come to appreciate and even understand the language better.
I'm starting this series (and hopefully more 😅🤞🏾) as a way to document and share my learnings.

What began as me writing an article about a new thing I'd learned turned into me trying to build something that at least 1 person would find useful. I ended up settling with a CLI tool aggregator which allows me customize my favorite CLI tools. One of things I was most curious about was building a database of my internet speeds over time.

From here on out, I'll be referring to the CLI tool we'll be building in this series as **"Commander"** and using it from the CLI by typing **cmdr** (I'm shortening this for convenience sake).

**Here's what I'd like to be able to do with Commander:**

1. Run my favorite CLI commands using command verbs of my choosing. e.g instead of **`git pull`**, I can run **`cmdr gp`** instead.
2. Save the output of the CLI command I'm most interested in (**`speed-test`**) as a JSON file on my computer.
3. Add extra fields to this JSON output (**`dateTime`** and **`connectionType`**). I'd like to know the exact time I ran the command and if my internet connection was wired (ethernet cable) or wireless (WiFi).
4. Upload every run of this command to a cloud database. I'd probably do something uncomplicated like updating an online spreadsheet.
5. Build a live dashboard with the data.
6. _save the world_ (I wish it were that easy).

I'll be building my first objective using C#. The end result is a console application that can be packaged and installed as a NuGet Package. To make this less verbose, I'll assume this is not your first time using C# or the .NET CLI. If this is, please check out official documentation on how to setup your development environment if you'd like to follow along:

- [Set up your development environment](https://docs.microsoft.com/en-us/dotnet/csharp/tour-of-csharp/tutorials/local-environment)

> Note: For this project, I'm using Visual Studio 2022 with C# 10 which was a part of the .NET 6 release. [Read more on C# 10](https://devblogs.microsoft.com/dotnet/welcome-to-csharp-10/).

### Creating a hello world .NET tool {#create-hello-world-dotnet-tool}

<!-- And now comes an SVG icon - {{< svg "bi-cart4" >}} - with text behind it.. -->

**What is a .NET tool?**

> The .NET CLI lets you create a console application as a tool, which others can install and run. DotNet tools are NuGet packages that are installed from the .NET CLI. For more information about tools, check out the [.NET tools overview](https://docs.microsoft.com/en-us/dotnet/core/tools/global-tools).

I'll be creating different checkpoints for the development of commander. The first is a hello world .NET tool that uses the **`System.CommandLine`** package to handle our CLI "inputs". These inputs are called _arguments_. We'll talk more about this in the [System.CommandLine section](#using-systemcommandline-to-make-our-tool-do-more-than-print-hello-world)

We'll be able to give the tool an input (a name) and then it says hello back. This checkpoint is inspired by [this dotnet walkthrough ](https://github.com/dotnet/command-line-api/blob/main/docs/Your-first-app-with-System-CommandLine.md).

Here, I'm using the .NET CLI command line tool to setup a console application.

```shell
dotnet new console -o commander
cd commander
dotnet add package System.CommandLine --prerelease
```

- The **`dotnet new`** keyword sets up a sample console application.
- The **`dotnet add`** keyword adds the **System.CommandLine** package to our project. The package is still in _Beta_ hence the **`prerelease`** flag.

After running the commands above, you'll have a simple application that prints out **`Hello, World!`** when you run the application.

Next, we want to start building out commander. This involves:

###### {#involves}

1. Making our project packable a.k.a we want the output of our project to be a NuGet package.
2. Install this package globally on our computer i.e running **`cmdr`** should print out **`Hello, World`**.
3. Use System.CommandLine to accept a name (**`--name`**) as input so we can print **`Hello, Mercy!`** if the `--name` input is included in the command.

**For the first step our `commander.csproj` file needs to look like this:**

```csharp
<Project Sdk="Microsoft.NET.Sdk">

	<PropertyGroup>
		<OutputType>Exe</OutputType>
		<TargetFramework>net6.0</TargetFramework>
		<ImplicitUsings>enable</ImplicitUsings>
		<Nullable>enable</Nullable>
		<PackAsTool>true</PackAsTool>
		<GeneratePackageOnBuild>true</GeneratePackageOnBuild>
		<ToolCommandName>cmdr</ToolCommandName>
		<PackageId>Commander</PackageId>
		<Description>Commander is a CLI tool aggregator that allows me customize my favorite CLI commands.</Description>
		<Authors>Mercy Markus</Authors>
	</PropertyGroup>

	<ItemGroup>
		<PackageReference Include="System.CommandLine" Version="2.0.0-beta3.22114.1" />
	</ItemGroup>

</Project>

```

From the official definitions:

- **PackAsTool** indicates whether the NuGet package should be configured as a .NET tool suitable for use with **`dotnet tool install`** (this is the command for installing .NET tools). It's what we are most interested in here.
- **GeneratePackageOnBuild** is for convenience sake. I'd like a new `.nupkg` (NuGet Package) to be generated every time I build the project. The alternative is running the **`nuget pack`** command manually every time.
- **ToolCommandName** specifies the command that'll invoke the tool after it's installed.
- **PackageId** is a case-insensitive NuGet package identifier, which must be unique across nuget.org or whatever gallery the NuGet package will reside in. IDs may not contain spaces or characters that are not valid for a URL, and generally follow .NET namespace rules. This is important when we're publishing our NuGet package to a gallery so it can be easily discovered and used.
- **Description** is a long description of the NuGet package for UI display.

**For the second step:**

The tool can be installed globally with: **`dotnet tool install --global --add-source ./bin/Debug Commander`**.

By default, NuGet attempts to find the package in package sources we've already added to the NuGet Package Manager. The `--add-source` flag's value points to the location of the NuGet package that gets generated when we build the project.

The output of running this is:

```shell
You can invoke the tool using the following command: cmdr
Tool 'commander' (version '1.0.0') was successfully installed.
```

Because we installed the tool globally, running `cmdr` from any terminal window will print `Hello, World!`.

More information can be found here: [How to manage .NET tools](https://docs.microsoft.com/en-us/dotnet/core/tools/global-tools#install-a-global-tool).

**For the third step:**
The following are happening:

1. We're using System.CommandLine to create an option (**`--name`**) that we can use to accept an input. We're also making this option a requirement when its parent command is invoked.
   > When an option is required and its parent command is invoked without it, an error occurs.
2. We're also setting the default value **`SetDefaultValue`** to **`World`**. This is to maintain the out-of-box experience we got before we [started building out commander](#start).
   Option: It's a symbol defining a named parameter and a value for that parameter.

```csharp {linenos=table}
// See https://aka.ms/new-console-template for more information
using System.CommandLine;

// Create name option:
var nameOption = new Option<string>(
    new[] { "--name", "-n" },
    description: "A 'name' option whose argument is a string representing a name.");

nameOption.IsRequired = true;
nameOption.SetDefaultValue("World");

var rootCommand = new RootCommand()
{
    nameOption
};

// rootCommand.AddOption(nameOption);

rootCommand.Description = "A Hello Greeter App";

rootCommand.SetHandler((string name) =>
{
    Console.WriteLine($"The value for --name is: {name}");
    Console.WriteLine($"Hello, {name}!");
}, nameOption);

// Parse the incoming argument and invoke the handler
return rootCommand.Invoke(args);
```

If you navigate to, you'll find an, you can share this exe wi
`~\commander\bin\Debug\net6.0`

### Using System.CommandLine to make our tool do more than print Hello world {#more-than-hello-world}

Accept command line arguments
Options, extensions, rootcommand, commands, commandHandler, aliases
