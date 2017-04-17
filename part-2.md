# Part 2 - Modernizing .NET apps - the platform

In this section we have an existing app, already packaged as an MSI. We'll Dockerize a few versions of the app, seeing how to do service updates and the benefits of Dockerfiles over MSIs.

## Steps

* [1. Package an ASP.NET MSI as a Docker image](#1)
* [2. Update the ASP.NET site with a new image version](#2)
* [3. Use Docker to build the source and package without an MSI](#3)

## <a name="1"></a>Step 1. Package an ASP.NET MSI as a Docker image

It's easy to package an MSI into a Docker image - use `COPY` to copy the MSI into the image, and `RUN` to install the application using `msiexec`, which is already bundled in the Windows base image.

Version 1.0 of our demo app is ready to go - check out the [Dockerfile](part-2/v1.0/Dockerfile). 

Packaging the app with Docker is the same `build` command - you'll use an image tag to identify the version:

```
cd C:\scm\github\sixeyed\dc-mta-workshop\part-2\v1.0
docker image build --tag $Env:dockerID/mta-app:1.0 .
```

The app uses SQL Server, but rather than start individual containers, you'll use [Docker Compose](https://docs.docker.com/compose/) to organize all the parts of the solution.

```
cd C:\scm\github\sixeyed\dc-mta-workshop\app
docker-compose -f docker-compose-1.0.yml up -d
```

Compose will start the SQL Server container and then the web app container. Get the web app's IP address and browse to it:

```
$ip = docker inspect --format '{{ .NetworkSettings.Networks.nat.IPAddress }}' app_mta-app_1
start "http://$ip/ProductLaunch"
```

Click on the 'Sign up' buton and register your details - that will write a row to SQL Server. The SQL Server container is only available to other Docker containers because no ports are published. You can execute PowerShell in the container to see your data:

```
docker exec app_mta-db_1 powershell "Invoke-SqlCmd -Query 'SELECT * FROM Prospects' -Database ProductLaunch"
```

Version 1.0 has a pretty basic UI. Next you'll upgrade to a fancier version.

## <a name="2"></a>Step 2. Update the ASP.NET site with a new image version

For the new app version there's a new MSI. The [Dockerfile](part-2/v1.1/Dockerfile) is exactly the same as v1.0, just using a different MSI. This scenario is where you have a new application release but you want to keep the same wunderlying Windows version.

Build the new app version:

```
cd C:\scm\github\sixeyed\dc-mta-workshop\part-2\v1.1

docker image build --tag $Env:dockerId/mta-app:1.1 .
```

And use Docker Compose to upgrade the solution:

```
cd C:\scm\github\sixeyed\dc-mta-workshop\app
docker-compose -f docker-compose-1.1.yml up -d
```

You'll see in the output that compose checks the current state of the application resources to the desired state in the YAML file. Here the SQL server container definition hasn't changed, so only the web application container is replaced.

The new container is running the new app version, but it has the same name as the prvious one. Check out the new app version by browsing to the container address:

```
$ip = docker inspect --format '{{ .NetworkSettings.Networks.nat.IPAddress }}' app_mta-app_1
start "http://$ip/ProductLaunch"
```

Sign up with another set of details, and when you repeat the SQL query you'll see that the new data is there with the original data:

```
docker exec app_mta-db_1 powershell "Invoke-SqlCmd -Query 'SELECT * FROM Prospects' -Database ProductLaunch"
```

The app is looking better, but the Dockerfile isn't very useful. It should describe everything that's needed to package the app, but most of the work is done in the opaque MSI file - built from this hug [Wix script](src/ProductLaunch/ProductLaunch.Web.Setup/Product.wxs).

Next you'll see how to package the same content in a different way, and upgrade the app container to a new version of Windows at the same time.

## <a name="3"></a>Step 3. Use Docker to build the source and package without an MSI

Docker 17.05 supports multi-stage builds, which we use now. The [Dockerfile](part-2/v1.2/Dockerfile) builds two images. The first uses a generic MSBuild image to build the Web project. The second packages the build output into an application image.

The Dockerfile approach removes the need for an MSI, or any build pre-requisites. Anyone with Docker can build and run the application, you don't need Visual Studio or even .NET installed on your machine.

It's also clear from the Dockerfile exactly how the app is built and installed, and this new version has a `HEALTHCHECK` which is good practice for production workloads.

Multi-stage builds run in the same way, but only the final image is tagged:

```
cd C:\scm\github\sixeyed\dc-mta-workshop

docker build -t $Env:dockerId/mta-app:1.2 -f part-2\v1.2\Dockerfile .
```

> Note that this image gets built from the base directory. That's so the full `src` folder is available to the build agent.


As before, upgrade the running application using Docker Compose:

```
cd C:\scm\github\sixeyed\dc-mta-workshop\app

docker-compose -f docker-compose-1.2.yml up -d
```

> Now is a good time to tell a story (until the healthcheck kicks in).

Browse to the app, and you'll see the UI is the same, but the deployment process is much cleaner - and we've upgraded Windows to get the latest hotfixes:

```
$ip = docker inspect --format '{{ .NetworkSettings.Networks.nat.IPAddress }}' app_mta-app_1
start "http://$ip/ProductLaunch"
```

The UX is the same too, so when you sign up you'll see a new set of details in the SQL container:

```
docker exec app_mta-db_1 powershell "Invoke-SqlCmd -Query 'SELECT * FROM Prospects' -Database ProductLaunch"
```

## Wrap Up

That's it for Part 2. In [Part 3](part-3.md) we'll modernize the app architecture, making use of the Docker platform to break features out of the monolith, and run them in lightweight containers.
