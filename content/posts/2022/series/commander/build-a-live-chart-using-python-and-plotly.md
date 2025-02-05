---
title: 'Building a CLI Tool Aggregator with C#: Plotly Live Interactive Chart using Python'
date: 2022-08-27T12:06:03+01:00
draft: false
description: "The last post in the 'commander' series. We're going to wrap up our CLI tool aggregator and build a live chart of our speed test results"
series: ['Commander']
tags: [Dotnet, Node, Python, Plotly]
---

{{< toc >}}

The main point of this series was to document my learnings from building DotNet global tools using C#. Building a CLI tool aggregator was a fun way to write about this.

### CLI Tool Aggregation

From the first post, **`commander`** sought to be a CLI tool that'll customize my favourite CLI commands and if you've followed the series from the beginning, you'd see that my primary focus was on the **speed test** command.

I was most curious about building a database of my internet speeds over time. Adding other commands to the tool was a bonus.

Below, we're adding 2 more commands to **`commander`**. The first is the **`wifi`** command which we'll use to check our WiFi password and the second is the **`ip`** command we'll be using to check our public IP address.

I found the WiFi command useful because I hardly ever remember my password when guests ask to connect to the WiFi and finding it on a Windows device involves [too many clicks.](https://www.lifewire.com/find-wifi-password-on-windows-11-5216845)

```Program.cs
var httpsOption = new Option<bool>(new[] { "--https", "-s" }, getDefaultValue: () => false, "Use DNS over HTTPS (DoH) instead of DNS?");

<speedCommand from last post goes here>

var wifiPasswordCommand = new Command("wifi", "checks your wifi password")
{
    Handler = CommandHandler.Create(() =>
        CommandRunner($"(npm list --global wifi-password-cli || npm install --global wifi-password-cli) && wifi-password", baseWorkingDirectory)),
};

var publicIpCommand = new Command("ip", "checks your public IP address")
{
    Handler = CommandHandler.Create<bool>((https) =>
    {
        if (https)
        {
            CommandRunner($"(npm list --global public-ip-cli || npm install --global public-ip-cli) && public-ip --https", baseWorkingDirectory);
        }
        else
        {
            CommandRunner($"(npm list --global public-ip-cli || npm install --global public-ip-cli) && public-ip", baseWorkingDirectory);
        }
    }),
};

cmdrRootCommand.AddCommand(wifiPasswordCommand);
cmdrRootCommand.AddCommand(publicIpCommand);

publicIpCommand.AddOption(httpsOption);
```

There's nothing new happening above. We're using the **`CommandRunner`** method we created in the last post to check if either global Node package has been installed. If they're not, we install them and run the respective commands attached to the command.

In the **`publicIpCommand`**, we're using the **`httpsOption`** boolean option to toggle between using the DNS IP lookup or [DNS over HTTPS](https://www.techtarget.com/searchsecurity/definition/DNS-over-HTTPS-DoH) IP lookup.

Finally, we add the commands to the root command and add the **`httpsOption`** to **`publicIpCommand`**.

### Import Speed Test Results to Google Sheets using gSpread and Python

I love the flexibility that comes with being able to run other program types thanks to the ability to create/run any process. This is why I'm able to use Python to handle the next sections.

I found a nifty library called gSpread that handles this well and is easy to configure.

There's some background work that needs to be done such as getting a Google Developers account, enabling the Google Sheets & Google Drive APIs, creating a service account and granting it access to an empty Google Sheet we'll be using to store our data.

[This medium post](https://medium.com/craftsmenltd/from-csv-to-google-sheet-using-python-ef097cb014f9) helped me do these.

> I'm also assuming you're a little familiar with python and know how to set up a virtual python environment and can use pip to install packages in said environment.

> You can read more about creating Python virtual environments [here](https://realpython.com/python-virtual-environments-a-primer/)

We'll be working with a file called **`upload.py`** which we'll use to upload our data to Google Sheets and build our Plotly chart.

```upload.py
import gspread
from oauth2client.service_account import ServiceAccountCredentials

scope = ["https://spreadsheets.google.com/feeds", 'https://www.googleapis.com/auth/spreadsheets',
         "https://www.googleapis.com/auth/drive.file", "https://www.googleapis.com/auth/drive"]

credentials = ServiceAccountCredentials.from_json_keyfile_name('client_secret.json', scope)
client = gspread.authorize(credentials)

Sheet_name ="Speed"
spreadsheet = client.open(Sheet_name)

with open('speed-results.csv', 'r') as file_obj:
    content = file_obj.read()
    client.import_csv(spreadsheet.id, data=content)
```

The following are happening:

1. We import the **`gSpread`** and **`ServiceAccountCredentials`** python packages.
2. We authorize gSpread to use the credentials on the Google Drive and Google Sheets scope we've defined to read our local speed test results and import them to the Google Sheet we created earlier.

### Build a Live Interactive Plotly Chart using Python

I chose Plotly as my data visualizer because of how interactive and flexible Plotly charts are. Building charts with code as opposed to using specific software such as Tableau, Google Charts or Power Bi was also a selling point.

We'll be breaking this section up into 3 parts.

In the first part:

1. We're using gSpread to open the speed test results Sheet and then inserting it into a Pandas DataFrame. Having our data in a DataFrame makes it easier to work with.
2. We're renaming the column names so they're clear and descriptive when we plot our chart.
3. We're creating 2 separate DataFrames for wired and wireless speed test results so we can compare them against each other visually.

```upload.py
import pandas as pd
worksheet = spreadsheet.worksheet(Sheet_name)
df = pd.DataFrame(worksheet.get_all_records())

column_names =  {"connectionType":"Connection Type", "datetime":"Date",
                 "downloadSpeed":"Download Speed (Mbps)", "latency":"Latency (Milliseconds)", "uploadSpeed":"Upload Speed (Mbps)"}

df.rename(columns = column_names, inplace = True)

df_wired = df[df["Connection Type"] == "Wired"]
df_wireless = df[df["Connection Type"] == "Wireless"]
```

In the second part:

1. We're using Plotly to plot the results of both wired and wireless speeds on the y-axis (a.k.a we're using the left side of the y-axis for both wired and wireless results) and the dates the tests occurred on the x-axis.
2. The plots are scatterplots whose markers are configured with different colours to depict what they represent.
3. Also, we add labels and a title to our x and y axes and centre our plot to our liking.

```upload.py
from plotly.subplots import make_subplots

fig = make_subplots(specs=[[{"secondary_y": False}]])

fig.add_scatter(x=df_wired["Date"], y=df_wired["Download Speed (Mbps)"],
                marker_color="orange", name="Download Speed (Mbps) - Wired")

fig.add_scatter(x=df_wired["Date"], y=df_wired["Upload Speed (Mbps)"],
                marker_color="mediumpurple", name="Upload Speed (Mbps) - Wired")

fig.add_scatter(x=df_wireless["Date"], y=df_wireless["Download Speed (Mbps)"],
                marker_color="blue", name="Download Speed (Mbps) - Wireless")

fig.add_scatter(x=df_wireless["Date"], y=df_wireless["Upload Speed (Mbps)"],
                marker_color="green", name="Upload Speed (Mbps) - Wireless")

fig.update_yaxes(title="Speed (Mbps)")
fig.update_xaxes(title="Date")

fig.update_layout(title={ "y":0.9, "x":0.5, "xanchor": "center", "yanchor": "top"})

fig.update_layout(title_text="Speed Test History")
```

Here's a sample image of how this plot can look like:

![image of speed test history](https://res.cloudinary.com/mercymarkus-com/image/upload/v1661610797/SpeedTestHistory_kc7xxo.png)

Lastly, in the third part:

1. We import chart studio for authenticating to our Chart Studio account using our account’s API key.
2. We're saving the figure we've created above to our Plotly account.

```upload.py
import chart_studio
import chart_studio.plotly as ply

username = '<add-plotly-username>'
api_key = '<add-plotly-password>'
chart_studio.tools.set_credentials_file(username=username, api_key=api_key)


ply.plot(fig, file="Speed Test History")
```

### Wrapping Things Up

The last piece of the puzzle is integrating the **`upload.py`** python script into Commander.

Remember our **`Program.cs`** file from earlier? We'll be adding 2 more instances of **`CommandRunner`** to the if-block of **`speedCommand`**.

```Program.cs
<Other SpeedCommand commands go here>

CommandRunner($"Copy-Item -Path \"{combinedBaseDirectory($"{fileName}.csv")}\" -Destination \"{combinedBaseDirectory("python3env")}\"", baseWorkingDirectory);

CommandRunner(".\\activate && cd ../ && python upload.py", combinedBaseDirectory($"python3env\\Scripts"));
```

The first instance involves copying the file our speed results were saved into to a new location. This new location is a python virtual environment (python3env) I'm using to install the packages I need for the **`upload.py`** file.

The second instance of CommandRunner activates the virtual environment, goes back to the previous directory (where the **`upload.py`** file is) and runs the python file so that every time we run **`cmdr speed -s`**, we're also uploading our results to Google Sheets and creating a Plotly chart with this data.

**This final post marks the end of the commander series. I hope you enjoyed reading this as much as I've enjoyed hacking it together and writing about it.**

### Improvement Notes

There are things we could have done better or more efficiently. Some of these include:

1. Importing only new speed results to Google Sheets instead of the entire CSV or uploading the result directly to Google Sheets.
2. Cleaning up cmdr's output and making it less verbose.

### Check Point

- [Third checkpoint.](https://github.com/MercyMarkus/commander/tree/check-point-3)
