# Getting started with the product platform

## Overview

In this guide you will learn how to set up your local environment for developing the product platform (aka Birdleaf) for the first time. By the end of this tutorial you will have everything you need to build, run, and test Birdleaf services on your machine.

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


## Build the App

If you look through the project heirarchy, you should see that each of the repositories has a solution file at the top (`/birdleaf/BirdleafAuth/BirdleafAuth.sln`) and each of the services within has a project file (`/birdleaf/BirdleafAuth/src/Accounts.Service/Accounts.Service.csproj`). A solution is basically a workspace for multiple projects. Running `dotnet build BirdleafAuth.sln` will build every project in `BirdleafAuth` while honoring the dependencies of each project. Therefore, to avoid dependency conflicts between services in different repositories we need to restructure our app to have one common solution. To do that, you'll need to install a custom .NET tool.

  ```
  dotnet tool install BirdleafTool
  ```

Then, make sure you are in the top level directory containing each of the repositories and run:

  ```
  dotnet BirdleafTool.initialize-birdleaf
  ```

This command performs various housekeeping tasks to configure your project for local development but most relevantly for now is that it will created a top level `.sln` file. Now you can build everything together.

  ```
  dotnet build birdleaf/Birdleaf.sln
  ```

> Building the entire solution is slow but only needs to be done when dependencies change, like after you pull new commits from main. If you are iterating on a single project, it will automatically get built each time you run it or you can build it manually by pointing to its .csproj file (ex: `dotnet build Accounts.Service/Accounts.Service.csproj`)

## Run and Test a Service API

Each service relies on various applications like Redis for caching or Postgres for data to function properly. With [Docker containers](https://docs.docker.com/) you can conveniently host all the applications you need locally in a way that is a way that is identical to our Kubernetes pods in production.

[Install Docker](https://docs.docker.com/desktop/)

Each service has a `docker-compose.yaml` file located at `<service>/dev/local/docker/docker-compose.yaml`. This file configures all of the docker images your service needs. For example, if your service needs Redis and has `REDIS_PORT: 4444` defined in `appsettings.json`, the `docker-compose.yaml` file will have instructions to start a Redis container listening on port `4444` as well.

  ```
  docker compose -f <service>/dev/local/docker/docker-compose.yaml up
  ```

When you are finished, shut down with:

  ```
  docker compose down
  ```

Now you can try running a service.

  ```
  dotnet run <service-name>.Service/<service-name>.Service.csproj
  ```

Your service should now be running locally and listening for requests on `localhost:<port>`. You can find or change the port number in `appsettings.json` in the top level directory of your service. If your service makes requests to APIs is other services, it will by default point to the latest dev images on Kubernetes.


##Test the Service API

Birdleaf Services use [gRPC](https://grpc.io/). An easy way to invoke gRPC calls is by used [Postman](). 
1. Download and open Postman.
2. Click `Workspace > New > gRPC`.
3. In the URL box, enter the url of your running service (`localhost:<port>`)
4. Birdleaf services use service reflection, which allows clients like Postman to load a full service definition over RPC.
5. Click `Select a Method` and choose one from the dropdown.
6. The `Message` area should now be populated with a sample payload, also loaded from server reflection. Edit any fields as desired.
7. Click `Send` and view the Response Payload

> NOTE: If server reflection fails, you may need to load the service definition into Postman manually. You can find the Protobuf file for a service in at `<service>/data/dto/service.proto`. In Postman select `Service Definition > Import a .proto file` and paste the contents of service.proto. 



