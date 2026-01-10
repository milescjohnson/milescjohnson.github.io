# Getting started with the product platform

## Overview

In this guide you will learn how to set up your local environment for developing the product platform (aka Birdleaf) for the first time. By the end of this tutorial you will have everything you need to build, run, and test and develop Birdleaf services on your machine. 

## Setup

*  Birdleaf is organized into a few different GitHub repositories. Create a new directory and clone each of them into it:
    ```
      mkdir birdleaf
    ```
    ```
      git clone https://github.com/milescjohnson/birdleaf/birdleaf-auth.git ./birdleaf
    ```
    ```
      git clone https://github.com/milescjohnson/birdleaf/birdleaf-backend.git ./birdleaf
    ```
    ```
      git clone https://github.com/milescjohnson/birdleaf/birdleaf-data.git ./birdleaf
    ```
*  Download and install the [.NET 7 SDK](https://dotnet.microsoft.com/en-us/download/dotnet/7.0). You can verify the installation from the command line with:
   
    ```
     dotnet --version
    ```
    
*  Import the project into the IDE of your choice. For native .NET support, most people use one of these:
    *  [Rider]()
    *  [VS Code]()
    *  [Visual Studio]()

## Build the app

If you look through the project heirarchy, you should see that each of the repositories has a solution file at the top (`/Birdleaf/BirdleafAuth/BirdleafAuth.sln`) and each of the services within has a project file (`/Birdleaf/BirdleafAuth/src/Accounts.Service/Accounts.Service.csproj`). A solution is basically a workspace for multiple projects. Running `dotnet build BirdleafAuth.sln` will build every project in `BirdleafAuth` while honoring the dependencies of each project. Therefore, to avoid dependency conflicts between services in different repositories we need to restructure our app to have one common solution. To do that, you'll need to install a custom .NET tool.

  ```
  dotnet tool install BirdleafTool
  ```
  ```
  dotnet BirdleafTool.initialize-birdleaf
  ```

This command performs various housekeeping tasks to configure your project for local development but most relevantly for now is that you now have a top level `.sln` file. Now you can build everything together.

  ```
  dotnet build ./Birdleaf/Birdleaf.sln
  ```

> Building the entire solution is slow but only needs to be done when dependencies change, like after you pull new commits from main. If you are iterating on a single project, it will automatically get built each time you run it or you can build it manually by pointing to its .csproj file `dotnet build Accounts.Service/Accounts.Service.csproj`

## Run and test a service API

Each service relies on various applications like Redis for caching or Postgres for data to function properly. With [Docker containers](https://docs.docker.com/) you can conveniently host all the applications you need locally in a way that is a way that is identical to our Kubernetes pods in production.

[Install Docker](https://docs.docker.com/desktop/)

Now you can try running a service.

  ```
  dotnet run Accounts.Service/Accounts.Service.csproj
  ```

Your service should now be running locally and listening for requests on `localhost:<port>`. You can find or change the port number in `appsettings.json` in the top level directory of your service. If your service makes requests to APIs is other services, it will by default point to the latest dev images on Kubernetes.

Most services rely on 




