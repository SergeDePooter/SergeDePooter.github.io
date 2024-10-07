---
title: C# docker on Synology NAS
date: 2024-10-03 21:30:00
categories: [C#]
tags: [c#, synology, nas, docker]
---
# C# docker on Synology NAS

## Pre-requisites

### Synology
Check https://www.synology.com/en-eu/dsm/packages/ContainerManager to see if your Synology NAS is compatible to run Container Manager.

Go to your NAS, login and perform following steps
1. Go to Package Center
2. Search for 'Container Manager'
3. Install package
4. Make sure Web Station is enabled

### C#
In this example I'm using C# / Blazor and Visual Studio Code.
To make sure you have all the pre-requisites installed:
- https://visualstudio.microsoft.com/downloads/
- https://dotnet.microsoft.com/en-us/apps/aspnet/web-apps/blazor

Some additional extensions for Visual Studio Code might be necessary:
- .Net Install Tool
- Docker

Check the extensions window in Visual Studio Code!

## Creating the (default) Blazor project
1. Open Visual Studio Code
2. Open a new terminal: Terminal -> New Terminal
3. Navigate to the folder where you want to put your project using the 'cd' command
```Shell
cd <your folder here>
```
4. Next, use the dotnet command to create a new project
```Shell
dotnet new blazor -o BlazorApp
```
5. Navigate to your new created project using cd
```Shell
cd BlazorApp
```
6. Make sure your application runs using
```Shell
dotnet watch
```

If all goes well, the terminal should open your Blazor-app:
![Local blazor project](/assets/2024-10-03/Schermafbeelding%202024-09-26%20om%2021.09.14-1.png)
As you can see, I edited the text of the main page a little bit.

### Upload your project to Synology Nas
Next step is to copy your entire project to a location on your Synology NAS.
* Go to your project on your pc / mac
* Zip in the full project folder
* Login to your NAS
* Open File Station
* Go to a location of your choosing
* Select Upload (skip or overwrite)
* Select your zip-file
* Unpack your zip-file on the NAS

Make sure you have a setup like this:
![Project on NAS](/assets/2024-10-03/Schermafbeelding%202024-09-26%20om%2021.53.27-1.png)

### Config Container Manager
Container manager is an application for building and hosting virtual containers. 

Before we can build the solution through the container manager, we need 2 more files to config and upload:
- Dockerfile
- Compose.yaml

Brief explanation:
#### Dockerfile
This is a set of instructions on what needs to be build, copied, expose http(s) ports (internal) and start in the container. Make sure to save the file without any extension.
Ref: https://docs.docker.com/reference/dockerfile/

#### Compose.yaml
This is set of instructions to configure your docker application's services, networks, volumes, etc. Make sure to save the file as 'compose.yaml'.
Ref: https://docs.docker.com/reference/compose-file/ 

In this setup, I will provide you with a basic simple setup for both Dockerfile & compose.yaml but I will not explain in detail what is happening, only the necessary. For more information, see the referenced documentation.

Compose.yaml
```yaml
version: '3.4'

services:

  web:
    image: ${DOCKER_REGISTRY-}web
    build:
      context: .
      dockerfile: Dockerfile
    ports:
      - 86:86
```
Most important part in this file is the setup of the ports because this will be the ports that Synology Web Station will use to open a port on the NAS. More on this later.

Dockerfile
```Docker
FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS base
WORKDIR /app
EXPOSE 86
EXPOSE 443
ENV ASPNETCORE_URLS=http://*:86

FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src
COPY ["BlazorDemo/BlazorDemo.csproj", "BlazorDemo/"]
RUN dotnet restore "BlazorDemo/BlazorDemo.csproj"
COPY . .
WORKDIR "/src/BlazorDemo"
RUN dotnet build "BlazorDemo.csproj" -c Release -o /app/build

FROM build AS publish
RUN dotnet publish "BlazorDemo.csproj" -c Release -o /app/publish /p:UseAppHost=false

FROM base AS final
WORKDIR /app
COPY --from=publish /app/publish .
ENTRYPOINT ["dotnet", "BlazorDemo.dll"]
```
Save both files on your pc and upload them to your NAS.
Put the files next to the root folder of the Blazor project, like this:
![alt text](/assets/2024-10-03/Schermafbeelding%202024-09-26%20om%2022.17.22-1.png)
Note that everything of the application is inside the folder BlazorDemo and both the compose.yaml & Dockerfile are next to the root folder. I prefer to keep those separate.

#### Container Manager
Time to open de Container Manager application on your Synology Nas.

1. Go to Projects in Container Manager
![Start project](/assets/2024-10-03/Schermafbeelding%202024-09-29%20om%2020.29.47-1.png)
2. Select 'Create'
3. In General Settings, choose 'Set Path'
![Setting path](<Schermafbeelding 2024-09-29 om 20.30.06-1.png>)
4. Select the root folder of the project
![Selecting root folder](<Schermafbeelding 2024-09-29 om 20.31.11-1.png>)
Container Manager will recognize that there is a compose.yaml inside the folder. Choose the use the existing file instead of (re-)creating a new one.
5. Give the project a name
![Project name](<Schermafbeelding 2024-09-29 om 20.31.51-1.png>)
6. Container Manager will ask to setup an entry in the Web Station to expose the container to a port. We will skip this for now and go to 'Next'.
![Create entry in web station](<Schermafbeelding 2024-09-29 om 20.32.13-1.png>)
7. Create the project!
![Create project](<Schermafbeelding 2024-09-29 om 20.32.25-1.png>)
8. You will get another screen where the process will continue with all the steps in de Dockerfile
![Building the container](<Schermafbeelding 2024-09-29 om 20.32.38-1.png>)
9. If all goes well, the project will succeed its build and show up in your Container Manager.
![Succesfull build](<Schermafbeelding 2024-09-29 om 20.45.37-1.png>)

#### Setting up network
In step 6 of the previous part, we skipped creating a port exposure in Web Station. The reason for this is fairly easy: I never got it to work in that step of the process. Luckily, and after a lot of searching and trying, there is another way.

1. Stop your project.
![Stop project](<Schermafbeelding 2024-09-29 om 20.53.27-1.png>)
2. Go to the details of your project and select 'Blazordemo-web-1'
![Project details - 1](<Schermafbeelding 2024-09-29 om 20.54.00-1.png>)
3. Go to Settings
![Project details - 2](<Schermafbeelding 2024-09-29 om 20.54.47-1.png>)
4. Here is the part where it gets a bit confusing. We asked Container Manager to skip the creation during the creation of a new project. If we here select 'Set up web portal via Webstation' and at port 86, it will automatically add a record in section 'Port Settings'. Remove the allready existing line with 2-times port 86.Check image below.
![Adding webstation port](<Schermafbeelding 2024-09-29 om 20.55.05-1.png>)
5. Save the settings. After saving, Container Manager will ask you to set up the connection in Web Station.
![Save portal settings](<Schermafbeelding 2024-09-29 om 20.55.26-1.png>)
6. Configure the settings in Web Station like following.
![Port setting](<Schermafbeelding 2024-09-29 om 20.55.51-1.png>)
7. Start your project again in Container Manager. If all goes well, you can navigate to your project on a local port.
![Running project](<Schermafbeelding 2024-09-29 om 21.16.55-1.png>)
