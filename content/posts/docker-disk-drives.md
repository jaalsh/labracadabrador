---
title: 'Creating a disk drive in a Windows Docker container'
date: 2023-05-14
# weight: 1
# aliases: ["/first"]
tags: ['docker', 'windows containers', 'registry', 'dos devices']
author: 'Jamie Sharpe'
# author: ["Me", "You"] # multiple authors
showToc: true
TocOpen: false
draft: true
hidemeta: false
comments: true
description: 'How to create a disk drive with a specific drive letter in a Docker windows container and mount data to it'
disableShare: false
disableHLJS: false
hideSummary: false
searchHidden: false
ShowReadingTime: true
ShowBreadCrumbs: true
ShowPostNavLinks: true
ShowWordCount: true
ShowRssButtonInSectionTermList: true
UseHugoToc: true
cover:
    image: '<image path/url>' # image path/url
    alt: '<alt text>' # alt text
    caption: '<text>' # display caption under cover
    relative: false # when using page bundles set this to true
    hidden: true # only hide on current single page
---

## The goal

The goal was to create to a disk a drive with the letter D: inside the windows container and then mount files to that letter. This was necessary as the code running within the container expected files to be placed on the D: drive to read them.

Additionally the drives needed to be available for use immediately without running a command on the container once deployed and persist across restarts.

## The obvious/simple answer

The obvious answer would be to to change the code to read from a location on the C: drive, however this was not possible.

## The solution - Registry(DOS Devices)

It is possible to permanently map a drive using the DOS Devices mechanism via the registry.

In PowerShell this is done with the following command:

```PowerShell
RUN New-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Control\Session` Manager\DOS` Devices" -Name "D:" -Value "\??\C:\MyDriveData";
```

Now all we need to do is modify our DockerFile to execute the above command, for example if I was building creating a Docker image to run a .NET Framework 4.8 project my Dockerfile could look something like this:

```
FROM mcr.microsoft.com/dotnet/framework/sdk:4.8 AS build

FROM mcr.microsoft.com/dotnet/framework/aspnet:4.8 AS runtime
EXPOSE 80
EXPOSE 443

SHELL ["powershell", "-Command", "$ErrorActionPreference = 'Stop'; $ProgressPreference = 'SilentlyContinue';"]

RUN New-Item -ItemType Directory C:\MyDriveData

# Add registry drive letter mapping from C:\MyDriveData to D:

# escape=`

RUN New-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Control\Session` Manager\DOS` Devices" -Name "D:" -Value "\??\C:\MyDriveData";

# Add registry drive letter mapping from C:\MyDriveData to E:

# escape=`

RUN New-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Control\Session` Manager\DOS` Devices" -Name "E:" -Value "\??\C:\MyDriveData";

ENTRYPOINT ["w3svc"]
```

Once built you can then run the container and mount files to the C:\MyDriveData folder and they will be mirrored to drive letter.

For example:

```
docker run -d --name diskdrivesample --mount source=c\:drivedataonhostmachine,target=c:\MyDriveData {image_name}
```
