# 301 Redirect Module: Install into an Existing Sitecore Docker Setup

## Purpose

This document explains how to build and install the 301 Redirect Module into an existing Sitecore Docker environment.

This repository contains:

- a .NET Framework 4.8 Sitecore module assembly
- Sitecore include config files
- Sitecore package manifests under `App_Data/packages`

This repository does **not** contain the serialized Sitecore items needed for a clean automated item deployment. Because of that, installation into Docker is a two-part process:

1. build and layer the DLL/config into the Sitecore container images
2. install the Sitecore items into the CM instance

## What Gets Deployed

From this solution, the main runtime artifacts are:

- `bin/SharedSource.RedirectModule.dll`
- `App_Config/Include/SharedSource/SharedSource.RedirectModule.config`

Optional files in this repo:

- `App_Config/Include/SharedSource/SharedSource.RedirectModule.Serialisation.config`
- `App_Config/Include/SharedSource/a-UnicornRedirect.config`

Do not use the Unicorn files as-is in Docker unless you rework them. The Unicorn root path is hardcoded to a local machine path and is not container-safe.

## Compatibility

The project references `Sitecore.Kernel` version `10.0.0`, so the safest supported target is a Sitecore `10.0` Docker environment.

If you are running a later Sitecore container image, the module may still work, but you should treat that as a compatibility test rather than a guaranteed match.

## Prerequisites

Before starting, make sure you have:

- a working Sitecore Docker setup already running or buildable
- Windows container support enabled
- .NET SDK installed with support for restoring and building .NET Framework SDK-style projects
- access to the Sitecore NuGet feed configured in `NuGet.Config`
- a Sitecore CM container in your environment
- permissions to modify your Sitecore Docker image build files

## Repository Notes

Important files in this repository:

- `RedirectModule.csproj`
- `NuGet.Config`
- `App_Config/Include/SharedSource/SharedSource.RedirectModule.config`
- `App_Data/packages/301 Redirect Module-1.9.xml`

The project is a class library, not a standalone web app. You do not publish it like an ASP.NET site. You build the DLL and copy the module assets into Sitecore.

## Step 1: Build the Module

From the repository root:

```powershell
dotnet restore RedirectModule.sln --configfile NuGet.Config
dotnet build RedirectModule.csproj -c Release
```

Expected output:

- `bin/SharedSource.RedirectModule.dll`

If restore fails, verify that the Sitecore package feed in `NuGet.Config` is reachable and that your machine can authenticate if needed.

## Step 2: Collect the Files to Deploy

After the build completes, collect these files:

```text
bin/SharedSource.RedirectModule.dll
App_Config/Include/SharedSource/SharedSource.RedirectModule.config
```

These are the minimum files required for the module code to load in Sitecore.

## Step 3: Add the Module to Your Sitecore Docker Image

The recommended approach is to bake the module into your Sitecore CM and CD images.

### CM vs CD

Deploy to `CM` because:

- the module includes Content Editor warnings
- the module handles item move events
- package installation happens in CM

Deploy to `CD` as well if you want redirects to execute for public traffic. The redirect processor runs in the request pipeline, so CD instances need the DLL and config too.

### Example Dockerfile for CM

Create or update your CM Dockerfile so it copies the module files into the Sitecore web root.

```dockerfile
ARG BASE_IMAGE
FROM ${BASE_IMAGE}

COPY module/bin/SharedSource.RedirectModule.dll C:/inetpub/wwwroot/bin/
COPY module/App_Config/Include/SharedSource/SharedSource.RedirectModule.config C:/inetpub/wwwroot/App_Config/Include/SharedSource/
```

### Example Dockerfile for CD

```dockerfile
ARG BASE_IMAGE
FROM ${BASE_IMAGE}

COPY module/bin/SharedSource.RedirectModule.dll C:/inetpub/wwwroot/bin/
COPY module/App_Config/Include/SharedSource/SharedSource.RedirectModule.config C:/inetpub/wwwroot/App_Config/Include/SharedSource/
```

### Suggested Build Context Layout

A simple layout for your Docker build context might look like this:

```text
docker/
  build/
    cm/
      Dockerfile
    cd/
      Dockerfile
  module/
    bin/
      SharedSource.RedirectModule.dll
    App_Config/
      Include/
        SharedSource/
          SharedSource.RedirectModule.config
```

Copy the built DLL and config into that `module` folder before building the images.

## Step 4: Rebuild the Sitecore Images

From your Sitecore Docker solution root, rebuild the relevant images.

Example:

```powershell
docker-compose build cm
docker-compose build cd
```

Or, if your solution uses the newer Docker Compose command:

```powershell
docker compose build cm
docker compose build cd
```

If you only need authoring-side installation first, rebuild `cm` first and validate there before rebuilding `cd`.

## Step 5: Start the Environment

Start or restart the Sitecore environment:

```powershell
docker-compose up -d
```

or

```powershell
docker compose up -d
```

Wait for the CM instance to become healthy before proceeding.

## Step 6: Install the Sitecore Items

This repository does not include ready-to-use serialized items in source control, so building the DLL is not enough for a clean install.

The module also needs Sitecore items such as:

- templates
- rules
- module root items
- roles

### Source of the Items

The package manifest is present here:

- `App_Data/packages/301 Redirect Module-1.9.xml`

That manifest indicates the module includes both files and Sitecore items.

### Recommended Installation Method

Install the module package into the CM instance using the Sitecore Installation Wizard or your existing package-install process.

Because this repo contains the package manifest but not a ready-to-use serialized item set in source control, your practical options are:

1. use an existing Sitecore package for version `1.9`
2. recreate the package from a working Sitecore instance
3. serialize the required items yourself and add them to your container deployment process

### Important Note

Do not assume that copying only the DLL and config is enough. The assembly may load, but the module will not be fully usable until the Sitecore items are present.

## Step 7: Verify the Config is Loaded

After the CM container starts, verify that the include config is present inside the container:

```text
C:\inetpub\wwwroot\App_Config\Include\SharedSource\SharedSource.RedirectModule.config
```

Also verify the assembly exists:

```text
C:\inetpub\wwwroot\bin\SharedSource.RedirectModule.dll
```

Then confirm Sitecore starts without configuration or assembly loading errors.

## Step 8: Validate Functionality

In CM, verify the module-specific items exist in the content tree after package installation.

Expected areas include module and rules-related items under Sitecore system nodes.

Then test:

1. create or inspect redirect items
2. request a known redirect URL
3. confirm the expected redirect status is returned
4. if using CD, test through the CD endpoint too

## Optional: Unicorn-Based Deployment

This repository includes Unicorn config files, but they are not Docker-ready.

### Why It Is Not Ready

`App_Config/Include/SharedSource/a-UnicornRedirect.config` contains a hardcoded local serialization path similar to:

```text
C:\Projects\301RedirectModule\serialisation
```

That path will not work inside a container.

### If You Want to Use Unicorn in Docker

You would need to:

1. serialize the required Sitecore items into source control
2. change the Unicorn root path to a container-safe path
3. mount or bake that serialization into the container image
4. ensure Unicorn is already part of the Sitecore environment
5. sync the items after the CM instance starts

Unless your existing Docker setup already uses Unicorn for module deployment, package installation is the faster path for this repository.

## Recommended Deployment Strategy

For this repository, the lowest-risk installation path is:

1. build the module DLL
2. copy the DLL and include config into CM and CD images
3. rebuild the images
4. start the environment
5. install the Sitecore package into CM
6. validate redirects in CM and CD

This avoids depending on incomplete serialization assets.

## Troubleshooting

### Build fails on package restore

Possible causes:

- the Sitecore NuGet feed is unavailable
- network access to the Sitecore feed is blocked
- missing credentials or restricted package access

Check `NuGet.Config` and verify package restore on the build machine.

### Sitecore starts but redirects do not work

Possible causes:

- the DLL was copied to CM but not CD
- the config file was not copied into the image
- the Sitecore items were never installed
- the request is hitting a role that does not have the module deployed

### Content Editor features are missing

Possible causes:

- the package items were not installed
- you are checking a CD instance instead of CM
- the module roles or rules items are missing

### Unicorn configuration breaks startup

Possible cause:

- `a-UnicornRedirect.config` is using a hardcoded filesystem path that does not exist in the container

If you are not intentionally using Unicorn for this module, do not include that file in the container image.

## Minimal Checklist

- [ ] restore packages using `NuGet.Config`
- [ ] build `RedirectModule.csproj` in Release
- [ ] collect `SharedSource.RedirectModule.dll`
- [ ] collect `SharedSource.RedirectModule.config`
- [ ] copy both into CM image
- [ ] copy both into CD image if redirects must run on CD
- [ ] rebuild Sitecore images
- [ ] start containers
- [ ] install Sitecore package items into CM
- [ ] verify redirects work

## Example Summary

Build:

```powershell
dotnet restore RedirectModule.sln --configfile NuGet.Config
dotnet build RedirectModule.csproj -c Release
```

Bake into image:

```dockerfile
COPY module/bin/SharedSource.RedirectModule.dll C:/inetpub/wwwroot/bin/
COPY module/App_Config/Include/SharedSource/SharedSource.RedirectModule.config C:/inetpub/wwwroot/App_Config/Include/SharedSource/
```

Then:

```powershell
docker compose build cm cd
docker compose up -d
```

After startup:

- install the Sitecore package items in CM
- verify redirect behavior

## Final Notes

This repository is close to container-ready on the code side, but not fully automated on the Sitecore item side.

If you want a fully repeatable Docker installation, the next step is to replace manual package installation with one of these:

1. a serialized item deployment flow
2. an automated Sitecore package installation step
3. a container startup task that imports the required Sitecore items